# June 19, 2019

The error:  
```
File "./udf/ext_age_check.py", line 116, in <module>
    unit_link = download_csv(lookup_link)
  File "./udf/ext_age_check.py", line 24, in download_csv
    dump = pd.read_csv( url )
pandas.errors.EmptyDataError: No columns to parse from file
```

The code (about line 110 iin ext_age_check.py):  
```python
    #loop through each individual strat_name_id
    for match in strat_name_id:
        #hit the api to find unit_matches through /units route
        lookup_link = 'https://macrostrat.org/api/units?format=csv&strat_name_id=%s' %(str(match)) 
        print(lookup_link)
        unit_link = download_csv(lookup_link)
```

The problem:  
This link (which relies on "match," a strat_name_id) points to nothing... For example, copy/paste this into your browser:  
"https://macrostrat.org/api/units?format=csv&strat_name_id=73142"

### Digging deeper.
Alright, I think I have figured this out.  

```ext_strat_phrases``` uses the /defs/strat_names route in the Macrostrat API to match a strat phrase ("strat_phrase_check") to it's corresponding strat_name_id(s) in the Macrostrat database.

Then, in ```ext_age_check```, we use the /units route to match a found stratigraphic entity's strat_name_id back to that entity's age in the database.

The problem arises because /defs/strat_names and /units are ***not the same databases***. You can have a strat_name_id in /defs/strat_names that does not show up in the /units route. We can talk about why later if you are interested via Zoom.

I will try to illustrate this for you with a real example from the demo app:  
First, in ext_strat_phrases:
1. We found "Reagan Sandstone"
2. Look up the Reagan Sandstone:
  [https://macrostrat.org/api/defs/strat_names?strat_name=Reagan%20Sandstone](https://macrostrat.org/api/defs/strat_names?strat_name=Reagan%20Sandstone)
3. Get its strat_name_id(s) (a stratigraphic entity can have more than one!) -- Follow this ^ link and see there are **two** entries, with:
```
"strat_name_id":73142
```
and 
```
"strat_name_id":3398
```
(all of this happens around Line 200 in ext_strat_phrases)
4. Save the strat_name_ids as a string like "73142~3398" in the strat_phrases table. For example:  
```
strom_demo=# SELECT DISTINCT strat_phrase, strat_name_id FROM strat_phrases WHERE strat_phrase='Reagan Sandstone';
   strat_phrase   | strat_name_id
------------------+---------------
 Reagan Sandstone | 73142~3398
(1 row)
```
5. 

# June 26, 2020
Was working through the output of ext_strat_phrases.py and found a few recurring patterns that may not be preferable. The words "series" and "system" frequently appeared in the stratigraphic phrase. Some examples include:

* PENNSYLVANIAN SYSTEM Cherokee Group
* CAMBRIAN SYSTEM Eminence Dolomite
* Series-Gasconade Dolomite
* Series-Bachelor Formation

Not entirely sure how to address these occurrences or even if they need to be addressed but just wanted to put them out here.
