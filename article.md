
> This article is still WIP

Author: Emanuel Calvo - Pythian

# [HIPPA](https://en.wikipedia.org/wiki/Health_Insurance_Portability_and_Accountability_Act), [RDS](https://aws.amazon.com/rds/postgresql/) and FTS applied for searching on PostgreSQL

I stepped with a case that required some attention on privacy regulations (HIPPA)
and performance issues caused by intensive CPU usage on RDS.

Implementing encryption on RDS using PGP isn't trivial, and there are a couple of
warnings before you decide to go forward within raw encryption or PGP functions.
All the current code and implementation is a _Proof of concept_ and I strongly
suggest to do your own analysis depending on your scenario.

It isn't easy to handle sometimes the load of encrypting plus using FTS as the
casts required to accomplish that are kinda incompatible. IMHO encrypted file
systems are more than enough in most of the cases, but... RDS.

What happens if you want to store related information of your customer, related
to `descriptions` or information stored on other rows such `drug` or `patology`.
Not only that. You want to score the results because you care about who matches
the best.

Also, keep in mind that the current approach could have issues in a very heavy
write environments, mainly derived from the logic held by the trigger function.

In the current article, we are going to show both PGP and raw encryption functions
from the [pgcrypto](http://www.postgresql.org/docs/9.4/static/pgcrypto.html) extension.
Please take a time to read about this before proceed to its implementation in
production.

## What do we have here?

There are a couple of notes to do regarding what is implemented.

- Both raw and PGP functions are used.
- I considered a per-row key based and a per-table key based. That is, you can
  either implement a different key per row-groups (you can implement one key per
  row, but I'm not going that way for clarity sake) or per-table basis (faster,
  cleaner but probably more insecure).
- For educational purposes, I'm uploading the private key into a table called
  `keys`. However, you never -never- want to do this.
- Raw encryption functions return bytea, doing the job of casting a little bit more
   dirty that PGP functions.
- The map table, does not store information. However, you are going to execute your
   inserts against this one. You will be inserting text data and the trigger
   function will be casting, processing and encrypting the data into the corresponding
  table.
- The key used for this article does not have a secret passphrase, which you probably
  want to have in critical environments.
- HIPPA specifies that the encryption happens at communication layer. This is
  something I'll avoid in this article as RDS uses SSL by default.

The trick over here is to store unencrypted part of the [SSN](https://en.wikipedia.org/wiki/Social_Security_number).
4 digit isn't enough information for identifying purposes if stolen. Also, for searching
you probably want a local full text search database to avoid dealing with the
issues on searching over encrypted data.

Although, if you are not interested to search through SSH, you can store few
characters from the last name, making easier the indexing.




## Tools scope and RDS instance setup

> If you are familiarized with `awscli`, configuring it and creating instances, feel free to skip
> entirely this section.

The following steps are tested on Linux OS, for more information you can check
this [link](http://docs.aws.amazon.com/cli/latest/userguide/installing.html).

Installing the `awscli` is easy as:

```
pip install awscli
```

To create an account, you can use `aws iam create-user --user-name test`. However,
I won't recommend to have this kind of things automated from the same platform
where you do the deploys. I assume that if you are reading the current section,
you are not into AWS as much to automate-everything. So, I'll recommend you to go
through the web interface once, and create your user from the IAM section, grant
it enough permissions (`AmazonRDSFullAccess`) and download the credentials.
With the creds, you are able to fill up the configuration using the following:

```
aws configure
```

Choosing a default region can make your life easier, however you will see that
I always will declare the `--region` option in the commands.

> Available regions code can be found at [Regions list](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)

Now, let's create an RDS instance. Beware in mind that not all the regions accept
the classic EC2 security groups permissions, so for this particular test I will
use `us-east-1` region.

```
aws rds create-db-instance --db-instance-identifier dbtest1 \
--db-name dbtest --db-instance-class db.t1.micro --engine postgres \
--master-username dbtestuser --master-user-password <password> --engine-version 9.4.5 \
 --allocated-storage 5 --region us-east-1
```


NOTE 1:
> Always choose the last patched version  9.4.x. There is an option for RDS to
> upgrade the patch version always.

NOTE 2:
> us-east-1 supports classic security groups setting, which is the recommended
> to run tests like that, without setting up a VPC. If you already have a VPC,
> you can use whatever region you prefer.

Now, let's create a _too wide open_ security group. Unless you now exactly which
is your IP for accessing the machines, I'll suggest to go wide open for the test.
If you have the IP or the range, you can use CIDR/IP as `x.x.x.x/32` or whatever
range you have.

```
aws rds create-db-security-group --region us-east-1 \
--db-security-group-name pgtest --db-security-group-description "My Test Security Group for RDS postgres"

aws rds authorize-db-security-group-ingress  --db-security-group-name pgtest --cidrip 0.0.0.0/0
```
If the following error is prompted, you proabbly missed to add the corresponding
permissions over the IAM user to create RDS instances:
```
A client error (AccessDenied) occurred when calling the CreateDBInstance operation: User: arn:aws:iam::984907411244:user/emanuelASUS is not authorized to perform: rds:CreateDBInstance on resource: arn:aws:rds:sa-east-1:984907411244:db:dbtest
```

Wait until the intance come online, then try the following:

```
aws rds describe-db-instances --region us-east-1 --db-instance-identifier dbtest1 | grep -i -A3  endp
            "Endpoint": {
                "Port": 5432,
                "Address": "dbtest1.chuxsnuhtvgl.us-east-1.rds.amazonaws.com"
            },
```

It takes a bit until the instance is up and running, so take a coffee or a tea (
  or mate ) before move forward.


## Pre-key setup

Let's generate the GPG keys:

```
$ gpg --gen-key   # choose DSA and Elgamal

$ gpg --list-secret-keys
/home/emanuel/.gnupg/secring.gpg
--------------------------------
sec   2048D/E65FF517 2016-02-17
uid                  Emanuel (This is a test key for an article) <calvo@pythian.com>
ssb   2048g/5C1EA9AB 2016-02-17

$ gpg --export E65FF517 > dummyKeys/public.key
$ gpg --export-secret-keys E65FF517 > dummyKeys/private.key

```

In order to avoid to be redundant, please follow the steps of the [pgcrypto documentation](http://www.postgresql.org/docs/9.4/static/pgcrypto.html).

Let's create the keys table. This step is entirely a way to facilitate the process
explained in the current article and is totally a non-recommendable production
practice:

```
CREATE TABLE keys (
  keyid varchar(16) PRIMARY KEY,
  pub bytea,
  priv bytea
);
```

Now, we need to import the keys into the RDS server. For doing that we are going to
use the `psql-Large Object` utility.

```
dbtest=> \lo_import '/home/emanuel/dummyKeys/public.key' pubk
lo_import 16438
dbtest=> \lo_import '/home/emanuel/dummyKeys/private.key' privk
lo_import 16439
```


Let's insert those files directly into the `keys` table, getting the key id using
the `pgp_key_id` function provided by the pgcrypto extension.

```
INSERT INTO keys VALUES( pgp_key_id(lo_get(16438)) ,lo_get(16438), lo_get(16439));
```

NOTE 1:
> Both keys should return the same id from the `pgp_key_id` function. If that's not
> the case, you are dealing with keys that won't work together.

NOTE 2:
> The key used in the current example has an empty passphrase which is - you got it -
> a security suicide. You won't use keys protected by a passphrase.


## Data definition

As we are going to put all the encryption inside the function, I placed a mapping
table that will receive columns as text type, convert inside the trigger-returning function
and insert the encrypted data only in the destination table (`__person__` and `__person__PGP`).
If you look at the code bellow, you will see that both tables have both `bytea` and text data types.

We are going to use PGP encryption and raw encryption. The advantage of the PGP
functions is that it allow us to use text as datatype, which save us some casting
(and obviously adds another layer of security). Raw functions are discouraged to be
used, however still an easy way to encrypt data using a simple passphrase and
a selected encryption method.



```
create table __person__
  (id serial PRIMARY KEY,
   fname bytea,
   lname bytea,
   description bytea,
   auth_drugs bytea, -- This is an encrypted text vector
   patology bytea,
   _FTS bytea -- this is the column storing the encrypted tsvector
   );

create table __person__map
     (fname text,
      lname text,
      description text,
      auth_drugs text[], -- This is an encrypted text vector
      patology text,
      _FTS text -- this is the column storing the encrypted tsvector
      );
```


The main function used by the trigger, will assign a weight to each column. This
is something that will help to rank the searches on our queries. Most of the time
you don't want to put the name of the person with too much weight, however one
of the requirements of the project was to have the person's name in order to
identify and check the history of a person to approve or double check the current
order.

The function works in a very tricky way. When you insert data, the `_FTS` field
needs the secret passphrase used to encrypt the data. The passphrase won't be
stored, but it'll be used for encrypting the data of the `_FTS` column.

The encryption will occur inside the function. This is because if you encrypt at
INSERT time, the function will need to decrypt the values inside it, as follows:

```
 encrypt(
        (setweight(to_tsvector(convert_from(decrypt(NEW.fname, secret, method), conv)) , 'B' ) ||
         setweight(to_tsvector(convert_from(decrypt(NEW.lname, secret, method),conv)), 'A') ||
         setweight(to_tsvector(convert_from(decrypt(NEW.description, secret, method),conv)), 'C') ||
         setweight(to_tsvector(convert_from(decrypt(NEW.auth_drugs, secret, method),conv)), 'C') ||
         setweight(to_tsvector(convert_from(decrypt(NEW.patology, secret, method),conv)), 'D')
    )::text::bytea, secret,method) ;
```


Also, the auth_drugs vector needs to be parsed. You can end up with something like
this as general purpose:

```
# select string_agg(('{drug1,drug2}'::text[])[i], ' ')
        from generate_series(array_lower('{drug1,drug2}'::text[], 1),
                             array_upper('{drug1,drug2}'::text[], 1)) as i;
 string_agg  
-------------
 drug1 drug2
(1 row)
```

However, as we are using FTS, we'll let the parser do all this work automatically.
It is important to understand that FTS's parsers clear all the punctuation and
stop words in the processed string, so you don't add any redundant processing.  


```
CREATE OR REPLACE FUNCTION _func_name_FTS_insert() RETURNS "trigger" AS $$
DECLARE
  secret __person__._FTS%TYPE;
  NEW_MAP __person__%ROWTYPE;
  method text := 'aes';
  conv name := 'SQL_ASCII';
  parsedDrugs text;
BEGIN
  secret := NEW._FTS::bytea; -- Here you are, we take the secret, then we step over it.

  NEW_MAP._FTS := encrypt(
                    (setweight(to_tsvector(NEW.fname) , 'B' ) ||
                     setweight(to_tsvector(NEW.lname), 'A') ||
                     setweight(to_tsvector(NEW.description), 'C') ||
                     setweight(to_tsvector(NEW.auth_drugs::text), 'C') ||
                     setweight(to_tsvector(NEW.patology), 'D')
                    )::text::bytea, secret,method) ;
  -- Now we encrypt the rest of the columns
  NEW_MAP.fname := encrypt(NEW.fname::bytea, secret,method);
  NEW_MAP.lname := encrypt(NEW.lname::bytea, secret,method);
  NEW_MAP.auth_drugs := encrypt(NEW.auth_drugs::text::bytea, secret,method);
  NEW_MAP.description := encrypt(NEW.description::bytea, secret,method);
  NEW_MAP.patology := encrypt(NEW.patology::bytea, secret,method);
  NEW_MAP.id := nextval('__person___id_seq'::regclass);

  INSERT INTO __person__ SELECT (NEW_MAP.*);
RETURN NULL;
END;
$$
LANGUAGE plpgsql;

-- Creating the trigger
CREATE TRIGGER trigger_befInsRow_name_FTS
BEFORE INSERT ON __person__map
FOR EACH ROW
EXECUTE PROCEDURE _func_name_FTS_insert();
```

> If you urge to use this kind of functions, I would suggest to use other procedural
> languages with better performance such C, plperl, plv8, plpython.  

The following function returns a variable amount of drugs randomly ordered (used
in the insert further):

```
CREATE OR REPLACE FUNCTION get_drugs_random(int)           
 RETURNS text[] AS
$BODY$
SELECT array_agg(p.drug) FROM (
  SELECT t.b FROM regexp_split_to_table(
      'Acetaminophen
Adderall
Alprazolam
Amitriptyline
Amlodipine
Amoxicillin
Ativan
Atorvastatin
Azithromycin
Ciprofloxacin
Citalopram
Clindamycin
Clonazepam
Codeine
Cyclobenzaprine
Cymbalta
Doxycycline
Gabapentin
Hydrochlorothiazide
Ibuprofen
Lexapro
Lisinopril
Loratadine
Lorazepam
Losartan
Lyrica
Meloxicam
Metformin
Metoprolol
Naproxen
Omeprazole
Oxycodone
Pantoprazole
Prednisone
Tramadol
Trazodone
Viagra
Wellbutrin
Xanax
Zoloft', '\n') t(b)
    ORDER BY random() LIMIT $1
) p(drug);
$BODY$
LANGUAGE 'sql' VOLATILE;
```

## Data loading

Now, let's insert 50 records with a bunch of aleatory values:

```
TRUNCATE TABLE __person__;

INSERT INTO __person__map
SELECT
('{Romulo,Ricardo,Romina,Fabricio,Francisca,Noa,Laura,Priscila,Tiziana,Ana,Horacio,Tim,Mario}'::text[])[round(random()*12+1)],
('{Perez,Ortigoza,Tucci,Smith,Fernandez,Samuel,Veloso,Guevara,Calvo,Cantina,Casas,Korn,Rodriguez,Ike,Baldo,Vespi}'::text[])[round(random()*15+1)],
('{some,random,text,goes,here}'::text[])[round(random()*5+1)] ,
get_drugs_random(round(random()*10)::int),
('{Anotia,Appendicitis,Apraxia,Argyria,Arthritis,Asthma,Astigmatism,Atherosclerosis,Athetosis,Atrophy,Abscess,Influenza,Melanoma}'::text[])[round(random()*12+1)],
'secret' FROM generate_series(1,50) ;

```


> As this is a POC, drugs and patologies aren't related and they are only for
> example purposes.


Once we already inserted the rows, we can query the table over `_FTS` column so
we can see how it got stored.

```
SELECT convert_from(decrypt(_FTS, 'secret'::bytea,'aes'), 'SQL_ASCII')
FROM __person__ LIMIT 5;
                                                                       convert_from                                                                        
----------------------------------------------------------------------------------------------------------------------------------
 'amlodipin':8C 'argyria':11 'ativan':10C 'clonazepam':4C 'lisinopril':6C 'lorazepam':5C 'lyrica':9C 'omeprazol':7C 'ricardo':1B 'rodriguez':2A 'text':3C
 'asthma':7 'doxycyclin':6C 'naproxen':4C 'ortigoza':2A 'prednison':5C 'priscila':1B 'text':3C
 'amoxicillin':8C 'argyria':11 'atorvastatin':6C 'codein':10C 'goe':3C 'lexapro':4C 'lorazepam':7C 'metformin':9C 'naproxen':5C 'ortigoza':2A 'tiziana':1B
 'acetaminophen':3C 'anotia':6 'doxycyclin':4C 'fabricio':1B 'omeprazol':5C 'veloso':2A

(5 rows)
```

Of course, you don't want to output `tsvector` columns as they have a very readable
format. Instead, you'll build the query using the FTS capabilities in the `WHERE`
condition and project only those columns you are interested into.

Differently from MySQL FTS implementation, the FTS does not return the values in
rank order. To do that, you need to use the ts_rank* functions. In the following
example, we use the ts_rank using the `id` column to identify the order in the
result set.


With ranking:

```
SELECT id, convert_from(decrypt(lname, 'secret','aes'), 'SQL_ASCII') lname,
           convert_from(decrypt(fname, 'secret','aes'), 'SQL_ASCII') fname,
       ts_rank( convert_from(decrypt(_FTS, 'secret','aes'), 'SQL_ASCII')::tsvector , query ) as rank
  FROM __person__ , to_tsquery('Mario | Casas | (Casas:*A & Mario:*B) ') query
  WHERE
    convert_from(decrypt(_FTS, 'secret','aes'), 'SQL_ASCII') @@
       query
  ORDER BY rank DESC;
```


Result:

```
id |  lname  | fname  |   rank   
----+---------+--------+----------
27 | Casas   | Romina | 0.303964
43 | Casas   | Noa    | 0.303964
 4 | Cantina | Mario  | 0.121585
35 | Guevara | Mario  | 0.121585
41 | Guevara | Mario  | 0.121585
44 | Smith   | Mario  | 0.121585
```

Without ranking:

```
SELECT id, convert_from(decrypt(lname, 'secret','aes'), 'SQL_ASCII') lname,
           convert_from(decrypt(fname, 'secret','aes'), 'SQL_ASCII') fname
  FROM __person__ , to_tsquery('Mario | Casas | (Casas:*A & Mario:*B) ') query
  WHERE
    convert_from(decrypt(_FTS, 'secret','aes'), 'SQL_ASCII') @@
       query;
```

Result:

```
id |  lname  | fname  
----+---------+--------
 4 | Cantina | Mario
27 | Casas   | Romina
35 | Guevara | Mario
41 | Guevara | Mario
43 | Casas   | Noa
44 | Smith   | Mario
```



A very awesome tutorial about FTS for PostgreSQL can be found [here](http://www.sai.msu.su/~megera/postgres/fts/doc/appendixes.html).

Source for drugs list http://www.drugs.com/drug_information.html
Source for diseases https://simple.wikipedia.org/wiki/List_of_diseases
Getting started with GPG keys https://www.gnupg.org/gph/en/manual/c14.html
AWS command line tool https://aws.amazon.com/cli/
Discussion in the community mailing lis [here](http://postgresql.nabble.com/Fast-Search-on-Encrypted-Feild-td1863960.html)
