# Data Wrangling OpenStreetMap

A xml osm file was downloaded for a selected area from OpenStreetMap.org. This report details the data auditing and cleaning performed on the raw dataset. After the data is cleaned, it is transformed and stored in a Sqlite database. With the stored dataset, the data and the chosen area are further explored.

## Map
I am using Las Vegas map for the analysis.
## Data Auditing

After auditing the street and postal code data, I found the following issues with the data source. The las_vegas_audit.py script is used to audit and find below issues. 
1. Various street name abbreviations for Avenue, Road, Boulevard, Parkway and Street are present. 
2. Some of the street names have numbers at the end of the name, although this issue is small. For Ex., '1': set(['Spanish Ridge Ave., Suite 1']
3. Some postal codes have more than 5 digits and some have alphabets at the begenning. Except one, all postal codes start with 89, which are the starting digits of postal codes in Las vegas.

Below mapping dictionary is created to change the abbreviated street names. update_street_name function is used to clean street names.
mapping = { "St": "Street", 
            "St.": "Street",
            "Ave" : "Avenue",
            "Ave.": "Avenue",
            "Blvd": "Boulevard",
            "Blv": "Boulevard",
            "Blvd.": "Boulevard",
            "Rd.": "Road",
            "Rd": "Road",
            "E" : "East",
            "W" : "West",
            "S" : "South",
            "N" : "North"
            }
First five digits in postal code is extracted to clean the postal codes. update_postal_code function is used to extract first five digits of the postal code.

## Process XML elements and create csv

After cleaning street names and postal codes, XML elements are processed using las_vegas_csv.py script and below csv files are created. Using csv_db.py, data base tables have been created from csv files.

    las_vegas.osm ----> 215.7 MB
    las_vegas.db -----> 150 MB
    nodes.csv --------> 80.8 MB
    nodes_tags.csv ---> 2.9 MB
    ways.csv ---------> 7.9 MB
    ways_nodes.csv ---> 27.5 MB
    ways_tags.csv ----> 13 MB

## Data exploration

### Number of nodes

```sql
SELECT COUNT(*) FROM nodes;
```
936445
### Number of ways

```sql
SELECT COUNT(*) FROM ways;
```
127183
### Number of distinct users

```sql 
SELECT count(distinct uid) FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) users; 
```
1374
### Top 10 contributers

```sql
SELECT uid,count(*) AS num_contributions
FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) a
GROUP BY 1
ORDER BY num_contributions DESC
LIMIT 10;
```
uid|num_contributions
7225438|94274
1330847|71964
214103|60791
496923|44033
9520187|40580
147510|39261
9515186|35840
574654|30147
336460|27731
5288591|26594
## Street names with numbers in the end

We can see below that we have street names with numbers in them. These street names should be cleaned in order to improve the data quality.

```sql
SELECT value
FROM nodes_tags
WHERE key='street' AND value REGEXP '[[:digit:]]$'
GROUP BY 1
LIMIT 10;
```
Howard Hughes Parkway, Suite 500
Lindell #D722
North Hualapai Way #115
S Decatur Blvd #107
S Grand Canyon Dr #106
S. Valley View Blvd. Ste 10
South 4th Street, STE 500
South Jones Boulevard Suite 110
South Pecos Road Suite 103
Sunrise Ave #103
### Top Cities and Provinces

```sql
SELECT value,count(*) as num
FROM
(
  SELECT value FROM nodes_tags WHERE key='city' or key='province'
  UNION ALL
  SELECT value FROM ways_tags WHERE key='city' or key='province'
)
GROUP BY 1
ORDER BY 2;
```
value|num
Las Vegas|792
North Las Vegas|139
Henderson|19
Sunrise Manor|18
Paradise|13
Las Vegas, NV|4
Las Vagas|3
Spring Valley|2
las vegas|2
Las Vegas NV|1
Las Veggas|1
Nellis AFB|1
Nellis Air Force Base|1
Whitney|1
### Top amenities

```sql
SELECT value,count(*)
FROM nodes_tags
WHERE key='amenity'
GROUP BY 1
ORDER BY count(*) DESC
LIMIT 20;
```
restaurant|498
fast_food|315
fuel|304
fountain|273
place_of_worship|250
school|143
bar|98
cafe|76
bench|72
bank|68
toilets|55
parking_entrance|54
pharmacy|42
post_office|40
parking|35
loading_dock|34
waste_disposal|31
fire_station|30
car_wash|26
bicycle_rental|21
I'm surprised to not see hotels in the top list of amenities in Vegas. So, I dig deeper into the top keys.

### Count of keys

```sql
SELECT key,count(*)
FROM nodes_tags
GROUP BY 1
ORDER BY count(*) DESC
LIMIT 20;
```
highway|20036
power|7219
name|3638
noexit|3612
source|2995
amenity|2802
created_by|2559
barrier|2133
crossing|1692
natural|1499
type|1486
output:electricity|1362
method|1356
location|1355
street|1029
housenumber|1001
postcode|950
wikidata|903
brand|862
wikipedia|855
I don't see any other key that might give information about amenities. So, lets check top streets and postal codes.

### Top streets and postal codes

```sql
SELECT value,count(*)
FROM nodes_tags
WHERE key='street'
GROUP BY 1
ORDER BY count(*) DESC
LIMIT 20;
```
West Charleston Boulevard|94
West Sahara Avenue|87
South Las Vegas Boulevard|53
South Fort Apache Road|33
West Cheyenne Avenue|31
East Sahara Avenue|26
Festival Plaza Drive|26
South Nellis Boulevard|26
East Flamingo Road|23
West Flamingo Road|21
West Tropicana Avenue|21
Boulder Highway|20
North Nellis Boulevard|20
East Tropicana Avenue|19
East Desert Inn Road|17
North Rainbow Boulevard|17
Red Rock Resort|17
South Maryland Parkway|17
Fremont Street|16
South Rainbow Boulevard|12
We see some famous streets in the list.

```sql
SELECT value,count(*)
FROM nodes_tags
WHERE key='postcode'
GROUP BY 1
ORDER BY count(*) DESC
LIMIT 20;
```
89135|104
89109|93
89117|92
89101|57
89102|56
89121|55
89104|42
89147|37
89119|34
89146|33
89103|32
89108|32
89129|27
89128|22
89169|22
89115|20
89122|20
89138|19
89110|17
89030|16
### Top 10 cuisines

```sql
SELECT nodes_tags.value, COUNT(*) as num
FROM nodes_tags 
    JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='restaurant') i
    ON nodes_tags.id=i.id
WHERE nodes_tags.key='cuisine'
GROUP BY nodes_tags.value
ORDER BY num DESC
LIMIT 10;
```
mexican|50
american|44
pizza|37
italian|23
asian|16
steak_house|16
japanese|13
chinese|11
thai|11
burger|8
## Conclusion

Las Vegas dataset has been actively contributed by users. However, the dataset has a few data quality issues such as typos in street names, city & province names and postal codes.  

These typos could be reduced by implementing country and city specific rules. For example, users should not be allowed to enter postal codes more than five digits and it should start from 88 or 89 for Las Vegas. Another way to reduce typos could be to suggest correct city or street names in case a user is typing wrong spelling. However, these solutions should be quick to adapt to future changes in the street names and postal codes. This requires a great deal of effort from OpenStreetMap community when done in a world scale. If the community is not quick to adapt the content, it could block users from entering current information and would discourage them from future contribution.

As a further work, the Las Vegas map could be improved with information of famous, tourist locations. In order to clean the dataset further, regular expressions could be used to clean street names that have numbers at the end.
