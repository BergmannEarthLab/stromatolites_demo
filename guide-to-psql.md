# After running the stromatolite demo app...
You will have produced a set of tables in a PSQL database called "strom_demo." Let's explore what's in these tables.

You can access your "strom_demo" database via the command line in 1 of 2 ways:  

  1.  Through a normal terminal window, by typing:
  ``` 
  $ psql -d strom_demo 
  ```
  2. Through the Postgres client, by opening the elephant icon and double clicking on the "strom_demo" database icon.  

Either method should  get you to a command line prompt that looks like
```
strom_demo=#
```

From here, we can start to explore these tables.

### First, print a list of all of the tables the stromatolites_demo app created:  
```
strom_demo=# \d
```
The result should look like:
```
                         List of relations
 Schema |              Name              |   Type   |    Owner
--------+--------------------------------+----------+--------------
 public | age_check                      | table    | JuliaWilcots
 public | bib                            | table    | JuliaWilcots
 public | refs_location                  | table    | JuliaWilcots
 public | results                        | table    | JuliaWilcots
 public | results_result_id_seq          | sequence | JuliaWilcots
 public | strat_dict                     | table    | JuliaWilcots
 public | strat_phrases                  | table    | JuliaWilcots
 public | strat_target                   | table    | JuliaWilcots
 public | strat_target_distant           | table    | JuliaWilcots
 public | strom_sentences_nlp352         | table    | JuliaWilcots
 public | target_adjectives              | table    | JuliaWilcots
 public | target_instances               | table    | JuliaWilcots
 public | target_instances_target_id_seq | sequence | JuliaWilcots
(13 rows)
```
(with your username replacing "JuliaWilcots")

At this point, it's good to match up the tables (age_check,  strat_dict, etc.) with where they are created in run.py and the ext_* files in the /udf/ directory.  

### Example 1
Let's explore ```strat_phrases```, which seeks to identify stratigraphic names (like "Beck Springs Dolomite").   

First, wee what columns are in the ```strat_phrases``` table:
```
strom_demo=# \d strat_phrases;
```
Then, let's cheks




Let's explore ```strat_target```, which will contain sentences that have both a *target* ("stromatol-") and a strat name (e.g. "Gamuza Fm."). 

### First, see what columns are in the ```strat_target``` table:
```
strom_demo=# \d strat_target;
```
### Check to see that the app is correctly identifying strat_names by looking at the root (e.g. "Gamuza") and flag (e.g. "Formation") for 10 distinct, random strat phrases:
```
strom_demo=# SELECT DISTINCT strat_phrase, strat_phrase_root, strat_flag FROM strat_phrases LIMIT 10;
```
It is also common to use this syntax in SQL:
```
strom_demo=# SELECT DISTINCT strat_phrase, strat_phrase_root, strat_flag 
strom_demo-# FROM strat_phrases 
strom_demo-# LIMIT 10;
```
You'll see somethting like:
```
        strat_phrase        | strat_phrase_root | strat_flag
----------------------------+-------------------+------------
 Musgravetown Group         | Musgravetown      | Group
 Chambersburg Limestone     | Chambersburg      | Limestone
 The Salanygol Formation    | The Salanygol     | Formation
 The Bonneterre Formation   | The Bonneterre    | Formation
 El Arpa Formation          | El Arpa           | mention
 Death Canyon Member        | Death Canyon      | Member
 Stephen Formation          | Stephen           | Formation
 Cambrian Burgess Shale     | Cambrian Burgess  | Shale
 The Chattanooga Shale      | The Chattanooga   | Shale
 Cambrian Baoshan Formation | Cambrian Baoshan  | Formation
(10 rows)
```
***We have found some mistakes!***  

Line 161 of ```ext_strat_phrases.py``` *should* be excluding words like "The" and "Cambrian" from strat_phrases. But, the results above show that it's not working properly! (I think it's likely this mistake is just a result of new syntax in Python 3 and the original app was written in Python 2.)

**WE NEED TO FIX THIS!!**

Looking at the next 10 strat_phrases, we see some more interesting (and unwanted) results:
```
strom_demo=# SELECT DISTINCT strat_phrase,strat_phrase_root, strat_flag FROM strat_phrases LIMIT 20;
 Aisciai Group                       | Aisciai                   | Group
 Autochthonous Mortinsburg Formation | Autochthonous Mortinsburg | Formation
 The Goodwin Limestone               | The Goodwin               | Limestone
 Howell Limestone                    | Howell                    | Limestone
 Cement Limestone                    | Cement                    | Limestone
 The Arrojos Formation               | The Arrojos               | Formation
 Chatsworth Limestone                | Chatsworth                | Limestone
 Middle Cambrian Spence Shale        | Middle Cambrian Spence    | Shale
 Mural Formation                     | Mural                     | Formation
 Middle Cambrian Wheeler Shale       | Middle Cambrian Wheeler   | Shale
 ```
 A few things to note:
  - "Autochthonous" is a term geologists use to mean "formed in place" -- it's an adjective rather than an actual formation name. 
  - Ignoring the mistake that we shouldn't have included "Cambrian" in the Spence or Wheeler Shales, we can see that there is actually more precise age information contained in these sentences -- we know the Wheeler Shale and Spence Shale are *Middle* Cambrian in age, not just Cambrian. (Andrew, we'll want to be able to keep and record that information!)  
  
It might be helpful to also read the whole sentence from which these strat phrases were extracted. To get the sentence(s) where we found the Spence Shale:
```
strom_demo=# SELECT sentence FROM strat_phrases WHERE strat_phrase='Middle Cambrian Spence Shale';
```
The result
```
                                                                       sentence
------------------------------------------------------------------------------------------------------------------------------------------------------
 Rigby -LRB- 1980 -RRB- described a large Vauxia from the Middle Cambrian Spence Shale Member of the Lead Bell Shale in northeastern Utah -LRB- loc .
(1 row)
```
is intriguing for a few reasons:
1. We notice that there are two strat flags ("Shale" and "Member"), which has confused the logic, which will stop at the first strat flag it sees in the sentence. My intuition is that it's probably not worth correcting the strat phrase ID logic, but is worth identifying double flags. We don't know if the Spence will be referred to as "Spence Shale" or "Spence Member" or "Spence Shale Member" in other papers -- it could be any of those options and we would want to recognize those as the same units!
2. There is a second strat phrase in this sentence ("Lead Bell Shale") (we can check with ```strom_demo=# SELECT sentence FROM strat_phrases WHERE strat_phrase='Lead Bell Shale';``` that it was also ID'd (it was).
3. There is stratigraphic heirarchy expressed in this sentence (the Spence Shale is a Member of the Lead Bell Shale)
4. There is **location information** in this sentence! We can infer that the Spence Shale and Lead Bell Shale are in northeastern Utah.
5. There is paleontological information ("large [Vauxia](https://en.wikipedia.org/wiki/Vauxia)") (the paleoDeepDive app would hopefully pick up on this).

  
### To do:
`- Fix line 161 in ext_strat_phrases so it does what it says it will do:
```
while (i-j)>(-1) and len(words[i-j])!=0 and words[i-j] != words[i-j+1] and words[i-j][0].isupper() and words[i-j] not in strat_flags and words[i-j].lower() not in stop_words and re.findall(r'\d+',  words[i-j])==[]:
```
- add "Autochthonous" to list of stop words (line 93 in ext_strat_phrases)
- add logic to find "Middle" in addition to "Cambrian"
