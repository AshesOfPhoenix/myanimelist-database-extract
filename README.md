# **Download data from MyAnimeList, clean it and load it into datasets**
6/7/2022
## The goal was to create a simple script that would extract and clean data from the MyAnimeList database and store it in CSV files.

* Data was extracted from the MyAnimeList database using the [MyAnimeList API](https://myanimelist.net/modules.php?go=api)
* Cleaned and transformed data was stored into CSV files
* Datasets were loaded onto the Kaggle website here: [Kaggle](https://www.kaggle.com/anime-extract)

### Prerequisites

To access the MyAnimeList API, you need to create an account on the website. After creating the account, you will need to obtain a token or rather Client ID from an API application which you can create in the API panel on your profile, [here](https://myanimelist.net/apiconfig). Visit the [MyAnimeList API](https://myanimelist.net/apiconfig/references/api/v2) to learn more about the API.

## Acknowledgments

References:

* [MyAnimeList API](https://myanimelist.net/apiconfig/references/api/v2)
* [Downloading MyAnimeList data for Google Data Analytics Capstone](https://github.com/patmendoza330/animelistextract)

## Usage

To generate a README.md file from Jupyter Notebook, use the following command:
* jupyter nbconvert --to markdown anime_extract.ipynb --output README.md


### Import

Insert obtained Client ID and specify folder to save datasets to into the code below.


```python
import pandas as pd
import numpy as np
import requests
import time
import ast

CLIENT_ID = 'YOUR_CLIENT_ID' # Client ID from MyAnimeList API
DATA_FOLDER = 'data/' # Folder to save data to
```

## MyAnimeList API

To learn how to use MyAnimeList API, visit the official API website: https://myanimelist.net/apiconfig/references/api/v2 and this GitHub page which contains an Unofficial API Specification: https://github.com/SuperMarcus/myanimelist-api-specification#search-anime .



```python
#data = requests.get('https://api.myanimelist.net/v2/anime/16498?fields=id,title,main_picture,alternative_titles,start_date,end_date,synopsis,mean,rank,popularity,num_list_users,num_scoring_users,nsfw,created_at,updated_at,media_type,status,genres,num_episodes,start_season,broadcast,source,average_episode_duration,rating,pictures,background,related_anime,related_manga,recommendations,studios,statistics&limit=4', headers={'X-MAL-CLIENT-ID': CLIENT_ID}) 
data = requests.get('https://api.myanimelist.net/v2/anime/ranking?ranking_type=all&limit=500', headers={'X-MAL-CLIENT-ID': CLIENT_ID}) 
print(data.json().keys()) 
data.json() 
print(data.json()['paging']['next']) # next page
```

    dict_keys(['data', 'paging'])
    https://api.myanimelist.net/v2/anime/ranking?offset=500&ranking_type=all&limit=500


## Loop through all ranked anime and add anime id and title to a pandas dataframe
* The API call has a limit of 500 anime per request.


```python
%%time
data = requests.get('https://api.myanimelist.net/v2/anime/ranking?ranking_type=all&limit=500', headers={'X-MAL-CLIENT-ID': CLIENT_ID}) # get all anime from ranking
df_anime_ids = pd.json_normalize(data.json()['data']).drop(['node.main_picture.medium', 'node.main_picture.large','ranking.rank'], axis=1) # get only anime ids and convert to dataframe
next = data.json()['paging']['next'] # get next page url
while next != None: # while there is a next page
    data = requests.get(next, headers={'X-MAL-CLIENT-ID': CLIENT_ID}) # get next page
    df_anime_ids = pd.concat([df_anime_ids, pd.json_normalize(data.json()['data']).drop(['node.main_picture.medium', 'node.main_picture.large','ranking.rank'], axis=1)], ignore_index=True) # concatenate dataframe and drop unnecessary columns
    try:
        next = data.json()['paging']['next'] # get next page url
    except:
        next = None # no more pages       
df_anime_ids.head()
```

    CPU times: total: 14.8 s
    Wall time: 3min 17s





<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>node.id</th>
      <th>node.title</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>5114</td>
      <td>Fullmetal Alchemist: Brotherhood</td>
    </tr>
    <tr>
      <th>1</th>
      <td>28977</td>
      <td>Gintama°</td>
    </tr>
    <tr>
      <th>2</th>
      <td>9253</td>
      <td>Steins;Gate</td>
    </tr>
    <tr>
      <th>3</th>
      <td>38524</td>
      <td>Shingeki no Kyojin Season 3 Part 2</td>
    </tr>
    <tr>
      <th>4</th>
      <td>9969</td>
      <td>Gintama'</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_anime_ids.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 20521 entries, 0 to 20520
    Data columns (total 2 columns):
     #   Column      Non-Null Count  Dtype 
    ---  ------      --------------  ----- 
     0   node.id     20521 non-null  int64 
     1   node.title  20521 non-null  object
    dtypes: int64(1), object(1)
    memory usage: 320.8+ KB


### List of all fields I want to extract about a specific anime from the MyAnimeList API


```python
fields = 'id,title,main_picture,alternative_titles,start_date,end_date,synopsis,mean,rank,popularity,num_list_users,num_scoring_users,nsfw,created_at,updated_at,media_type,status,genres,my_list_status,num_episodes,start_season,source,average_episode_duration,rating,pictures,background,related_anime,related_manga,recommendations,studios,statistics'
```

### Display a sample request to the API, which returns all information about the specified anime in json format
The data obtained contains all fields specified in the above cell


```python
data = requests.get('https://api.myanimelist.net/v2/anime/' + str(5114) + '?fields=' + fields, headers={'X-MAL-CLIENT-ID': CLIENT_ID})
data.json()
```




    {'id': 5114,
     'title': 'Fullmetal Alchemist: Brotherhood',
     'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/1223/96541.jpg',
      'large': 'https://api-cdn.myanimelist.net/images/anime/1223/96541l.jpg'},
     'alternative_titles': {'synonyms': ['Hagane no Renkinjutsushi: Fullmetal Alchemist',
       'Fullmetal Alchemist (2009)',
       'FMA',
       'FMAB'],
      'en': 'Fullmetal Alchemist: Brotherhood',
      'ja': '鋼の錬金術師 FULLMETAL ALCHEMIST'},
     'start_date': '2009-04-05',
     'end_date': '2010-07-04',
     'synopsis': 'After a horrific alchemy experiment goes wrong in the Elric household, brothers Edward and Alphonse are left in a catastrophic new reality. Ignoring the alchemical principle banning human transmutation, the boys attempted to bring their recently deceased mother back to life. Instead, they suffered brutal personal loss: Alphonse\'s body disintegrated while Edward lost a leg and then sacrificed an arm to keep Alphonse\'s soul in the physical realm by binding it to a hulking suit of armor.\n\nThe brothers are rescued by their neighbor Pinako Rockbell and her granddaughter Winry. Known as a bio-mechanical engineering prodigy, Winry creates prosthetic limbs for Edward by utilizing "automail," a tough, versatile metal used in robots and combat armor. After years of training, the Elric brothers set off on a quest to restore their bodies by locating the Philosopher\'s Stone—a powerful gem that allows an alchemist to defy the traditional laws of Equivalent Exchange.\n\nAs Edward becomes an infamous alchemist and gains the nickname "Fullmetal," the boys\' journey embroils them in a growing conspiracy that threatens the fate of the world.\n\n[Written by MAL Rewrite]',
     'mean': 9.14,
     'rank': 1,
     'popularity': 3,
     'num_list_users': 2892519,
     'num_scoring_users': 1843357,
     'nsfw': 'white',
     'created_at': '2008-08-21T03:35:22+00:00',
     'updated_at': '2022-04-18T05:06:13+00:00',
     'media_type': 'tv',
     'status': 'finished_airing',
     'genres': [{'id': 1, 'name': 'Action'},
      {'id': 2, 'name': 'Adventure'},
      {'id': 8, 'name': 'Drama'},
      {'id': 10, 'name': 'Fantasy'},
      {'id': 38, 'name': 'Military'},
      {'id': 27, 'name': 'Shounen'}],
     'num_episodes': 64,
     'start_season': {'year': 2009, 'season': 'spring'},
     'source': 'manga',
     'average_episode_duration': 1460,
     'rating': 'r',
     'pictures': [{'medium': 'https://api-cdn.myanimelist.net/images/anime/13/13738.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/13/13738l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/2/17090.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/2/17090l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/2/17472.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/2/17472l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/5/47603.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/5/47603l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/10/57095.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/10/57095l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/7/74317.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/7/74317l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/1521/94614.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/1521/94614l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/1208/94745.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/1208/94745l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/1223/96541.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/1223/96541l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/1286/96542.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/1286/96542l.jpg'},
      {'medium': 'https://api-cdn.myanimelist.net/images/anime/1629/108486.jpg',
       'large': 'https://api-cdn.myanimelist.net/images/anime/1629/108486l.jpg'}],
     'background': '',
     'related_anime': [{'node': {'id': 121,
        'title': 'Fullmetal Alchemist',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/10/75815.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/10/75815l.jpg'}},
       'relation_type': 'alternative_version',
       'relation_type_formatted': 'Alternative version'},
      {'node': {'id': 6421,
        'title': 'Fullmetal Alchemist: Brotherhood Specials',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/1493/91571.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/1493/91571l.jpg'}},
       'relation_type': 'side_story',
       'relation_type_formatted': 'Side story'},
      {'node': {'id': 7902,
        'title': 'Fullmetal Alchemist: Brotherhood - 4-Koma Theater',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/3/76154.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/3/76154l.jpg'}},
       'relation_type': 'spin_off',
       'relation_type_formatted': 'Spin-off'},
      {'node': {'id': 9135,
        'title': 'Fullmetal Alchemist: The Sacred Star of Milos',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/2/29550.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/2/29550l.jpg'}},
       'relation_type': 'side_story',
       'relation_type_formatted': 'Side story'}],
     'related_manga': [],
     'recommendations': [{'node': {'id': 11061,
        'title': 'Hunter x Hunter (2011)',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/1337/99013.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/1337/99013l.jpg'}},
       'num_recommendations': 101},
      {'node': {'id': 16498,
        'title': 'Shingeki no Kyojin',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/10/47347.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/10/47347l.jpg'}},
       'num_recommendations': 39},
      {'node': {'id': 1482,
        'title': 'D.Gray-man',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/13/75194.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/13/75194l.jpg'}},
       'num_recommendations': 23},
      {'node': {'id': 9919,
        'title': 'Ao no Exorcist',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/10/75195.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/10/75195l.jpg'}},
       'num_recommendations': 17},
      {'node': {'id': 38000,
        'title': 'Kimetsu no Yaiba',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/1286/99889.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/1286/99889l.jpg'}},
       'num_recommendations': 15},
      {'node': {'id': 1575,
        'title': 'Code Geass: Hangyaku no Lelouch',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/5/50331.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/5/50331l.jpg'}},
       'num_recommendations': 15},
      {'node': {'id': 2251,
        'title': 'Baccano!',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/3/14547.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/3/14547l.jpg'}},
       'num_recommendations': 15},
      {'node': {'id': 22199,
        'title': 'Akame ga Kill!',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/1429/95946.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/1429/95946l.jpg'}},
       'num_recommendations': 10},
      {'node': {'id': 23755,
        'title': 'Nanatsu no Taizai',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/8/65409.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/8/65409l.jpg'}},
       'num_recommendations': 9},
      {'node': {'id': 1735,
        'title': 'Naruto: Shippuuden',
        'main_picture': {'medium': 'https://api-cdn.myanimelist.net/images/anime/1565/111305.jpg',
         'large': 'https://api-cdn.myanimelist.net/images/anime/1565/111305l.jpg'}},
       'num_recommendations': 9}],
     'studios': [{'id': 4, 'name': 'Bones'}],
     'statistics': {'status': {'watching': '220764',
       'completed': '2095325',
       'on_hold': '98772',
       'dropped': '45327',
       'plan_to_watch': '432363'},
      'num_list_users': 2892551}}



## Loop through all anime ids from previous dataframe and extract anime information for each anime
* Filter out anime without mean score and rank (adult anime)
* Pause for 5 minutes every 500 requests to avoid being blocked


```python
%%time
print("Starting data aquasition for every anime ID in the list")
rq_limit = 500
print("Limited to " + str(rq_limit) + " sequential requests, otherwise server denies requests")

data = requests.get('https://api.myanimelist.net/v2/anime/' + str(df_anime_ids['node.id'][0]) + '?fields=' + fields, headers={'X-MAL-CLIENT-ID': CLIENT_ID}) # get json data for anime with myanimelist api
df_anime = pd.json_normalize(data.json()) # convert json to pandas dataframe

for cnt in range(1, len(df_anime_ids['node.id'])): # loop through all anime IDs
    data = requests.get('https://api.myanimelist.net/v2/anime/' + str(df_anime_ids['node.id'][cnt]) + '?fields=' + fields, headers={'X-MAL-CLIENT-ID': CLIENT_ID})
    try: # if the anime is found
        anim_json = data.json()  # get the json
        if(not np.isnan(anim_json['mean']) and not np.isnan(anim_json['rank'])): # if mean and rank are not nan
            #anim_json.pop('background', None) # remove background          
            df_anime = pd.concat([df_anime, pd.json_normalize(anim_json)], ignore_index=True)#.drop(['node.main_picture.medium', 'node.main_picture.large','ranking.rank'], axis=1)
    except:
        None
    if (cnt > 1 and cnt % 100 == 0): # print progress every 100 requests
        print(str(cnt) + " requests")
    if (cnt % rq_limit == 0):                           
        print("Waiting for 5 minutes before continuing") # pause the requests for 5 minutes, otherwise, the server will deny requests
        minutes = 6
        while (minutes > 1): # wait for 5 minutes
            print(str(minutes-1) + " minutes left")
            time.sleep(60)
            minutes -= 1
        print("Starting up again")
df_anime.head(5)
```


```python
df_anime.columns 
```




    Index(['mal_id', 'title', 'start_date', 'end_date', 'synopsis', 'mean', 'rank',
           'popularity', 'num_list_users', 'num_scoring_users', 'nsfw',
           'created_at', 'updated_at', 'media_type', 'status', 'genres',
           'num_episodes', 'source', 'average_episode_duration', 'rating',
           'pictures', 'background', 'related_anime', 'related_manga',
           'recommendations', 'studios', 'main_picture.medium',
           'main_picture.large', 'alternative_titles.synonyms',
           'alternative_titles.en', 'alternative_titles.ja', 'start_season.year',
           'start_season.season', 'statistics.status.watching',
           'statistics.status.completed', 'statistics.status.on_hold',
           'statistics.status.dropped', 'statistics.status.plan_to_watch',
           'statistics.num_list_users'],
          dtype='object')



### Save extracted anime information to a csv file


```python
df_anime.index.name = 'Index' # rename the index
df_anime.rename(columns={"id": "mal_id"}, inplace=True) # rename the id column to mal_id
df_anime = df_anime[df_anime['media_type'].isin(['tv', 'movie', 'ova', 'special', 'ona'])] # only include TV, Movie, OVA, Special, ONA
df_anime.reset_index(drop=True, inplace=True) # reset the index
df_anime.to_csv(DATA_FOLDER + 'anime_extract.csv', sep=';', encoding='utf-8') # save the dataframe as a csv file
df_anime.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 11613 entries, 0 to 11612
    Data columns (total 39 columns):
     #   Column                           Non-Null Count  Dtype  
    ---  ------                           --------------  -----  
     0   mal_id                           11613 non-null  int64  
     1   title                            11613 non-null  object 
     2   start_date                       11603 non-null  object 
     3   end_date                         11501 non-null  object 
     4   synopsis                         11410 non-null  object 
     5   mean                             11613 non-null  float64
     6   rank                             11613 non-null  int64  
     7   popularity                       11613 non-null  int64  
     8   num_list_users                   11613 non-null  int64  
     9   num_scoring_users                11613 non-null  int64  
     10  nsfw                             11613 non-null  object 
     11  created_at                       11613 non-null  object 
     12  updated_at                       11613 non-null  object 
     13  media_type                       11613 non-null  object 
     14  status                           11613 non-null  object 
     15  genres                           11576 non-null  object 
     16  num_episodes                     11613 non-null  int64  
     17  source                           10106 non-null  object 
     18  average_episode_duration         11613 non-null  int64  
     19  rating                           11514 non-null  object 
     20  pictures                         11613 non-null  object 
     21  background                       1507 non-null   object 
     22  related_anime                    11613 non-null  object 
     23  related_manga                    11613 non-null  object 
     24  recommendations                  11613 non-null  object 
     25  studios                          11613 non-null  object 
     26  main_picture.medium              11609 non-null  object 
     27  main_picture.large               11609 non-null  object 
     28  alternative_titles.synonyms      11613 non-null  object 
     29  alternative_titles.en            6178 non-null   object 
     30  alternative_titles.ja            11595 non-null  object 
     31  start_season.year                10969 non-null  float64
     32  start_season.season              10969 non-null  object 
     33  statistics.status.watching       11613 non-null  int64  
     34  statistics.status.completed      11613 non-null  int64  
     35  statistics.status.on_hold        11613 non-null  int64  
     36  statistics.status.dropped        11613 non-null  int64  
     37  statistics.status.plan_to_watch  11613 non-null  int64  
     38  statistics.num_list_users        11613 non-null  int64  
    dtypes: float64(2), int64(13), object(24)
    memory usage: 3.5+ MB


df_anime = pd.read_csv('data/anime_extract.csv', sep=';', encoding='utf-8')
df_anime.index.name = 'Index'
df_anime.drop(df_anime.columns[0], axis=1, inplace=True)
df_anime.rename(columns={"id": "mal_id"}, inplace=True)
df_anime = df_anime[df_anime['media_type'].isin(['tv', 'movie', 'ova', 'special', 'ona'])]
df_anime.reset_index(drop=True, inplace=True)
df_anime.info()

### Save anime ids and titles to a csv file


```python
df_anime_titles = df_anime[['mal_id', 'title']].copy() # copy the dataframe columns to a new dataframe
df_anime_titles.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_titles.index.name = "Index" # rename the index
df_anime_titles.to_csv(DATA_FOLDER + 'anime_titles.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_titles.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>title</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Cowboy Bebop</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>Cowboy Bebop: Tengoku no Tobira</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>Trigun</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>Witch Hunter Robin</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>Bouken Ou Beet</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime rankings to csv file


```python
df_anime_ranking = df_anime[['mal_id', 'mean', 'rank', 'popularity', 'rating', 'num_scoring_users']].copy() # copy the dataframe columns to a new dataframe
df_anime_ranking.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_ranking.index.name = "Index" # set the index name to "Index"
df_anime_ranking.to_csv(DATA_FOLDER + 'anime_ranking.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_ranking.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>mean</th>
      <th>rank</th>
      <th>popularity</th>
      <th>rating</th>
      <th>num_list_users</th>
      <th>num_scoring_users</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>8.76</td>
      <td>37</td>
      <td>42</td>
      <td>r</td>
      <td>1617259</td>
      <td>832701</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>8.38</td>
      <td>175</td>
      <td>566</td>
      <td>r</td>
      <td>334185</td>
      <td>192661</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>8.22</td>
      <td>310</td>
      <td>242</td>
      <td>pg_13</td>
      <td>659514</td>
      <td>328258</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>7.26</td>
      <td>2708</td>
      <td>1678</td>
      <td>pg_13</td>
      <td>105582</td>
      <td>41521</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>6.96</td>
      <td>4073</td>
      <td>4843</td>
      <td>pg</td>
      <td>14304</td>
      <td>6239</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime ratings to csv file


```python
df_anime_rating = df_anime[['mal_id', 'rating']].copy() # copy the dataframe columns to a new dataframe
df_anime_rating.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_rating.index.name = "Index" # set the index name to "Index"
df_anime_rating.to_csv(DATA_FOLDER + 'anime_rating.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_rating.head()
```

### Save anime dates to csv file


```python
df_anime_dates = df_anime[['mal_id', 'start_date', 'end_date', 'start_season.year', 'start_season.season']].copy() # copy the dataframe columns to a new dataframe
df_anime_dates.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_dates.index.name = "Index" # set the index name to "Index"

df_anime_dates.rename(columns={"start_season.year": "anime_season_year"}, inplace=True) # rename the column
df_anime_dates.rename(columns={"start_season_season": "anime_season"}, inplace=True) # rename the column
df_anime_dates["start_season_year"] = df_anime_dates["start_season_year"].fillna(0).astype(np.int64) # fill the NaN with 0 and convert to int64

df_anime_dates.to_csv(DATA_FOLDER + 'anime_dates.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_dates.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>start_date</th>
      <th>end_date</th>
      <th>start_season_year</th>
      <th>start_season.season</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1998-04-03</td>
      <td>1999-04-24</td>
      <td>1998</td>
      <td>spring</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>2001-09-01</td>
      <td>2001-09-01</td>
      <td>2001</td>
      <td>summer</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>1998-04-01</td>
      <td>1998-09-30</td>
      <td>1998</td>
      <td>spring</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>2002-07-03</td>
      <td>2002-12-25</td>
      <td>2002</td>
      <td>summer</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>2004-09-30</td>
      <td>2005-09-29</td>
      <td>2004</td>
      <td>fall</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime genres and demographics to csv files


```python
df_anime_genres = pd.DataFrame(columns=['mal_id', 'genre_id']) # create a dataframe with the columns
df_genres_d = pd.DataFrame(columns=['genre_id', 'genre_de']) # create a dataframe with the columns
df_anime_demographic = pd.DataFrame(columns=['mal_id', 'demo_id']) # create a dataframe with the columns
df_demographic_d = pd.DataFrame(columns=['demo_id', 'demo_de']) # create a dataframe with the columns

for row in df_anime.iterrows(): # iterate through the dataframe
    genres_str = row[1]['genres'] 
    if(pd.isna(genres_str)): # if the genres_str is NaN continue
        continue
    genres_str = '{"genres": ' + genres_str.replace("'id'", "'genre_id'").replace("name", "genre_de") + '}'
    genre_d = ast.literal_eval(genres_str) # convert the string to a dictionary
    genres_d = pd.json_normalize(genre_d['genres']) # normalize the json
    
    for genre in genre_d['genres']: # iterate through the genres and demographics
        if(genre['genre_id'] in [15, 25, 27, 42, 43]): # if the genre is in the list of demographics
            df_anime_demographic.loc[df_anime_demographic.shape[0]] = [row[1]['mal_id'], genre['genre_id']] # add the demographic to the dataframe
            if(genre['genre_id'] not in df_demographic_d['demo_id'].values): # if the demographic is not in the dataframe
                df_demographic_d.loc[df_demographic_d.shape[0]] = [genre['genre_id'], genre['genre_de']] # if the demographic not already in add it to the dataframe
        else: # if the genre is not in the list of demographics
            df_anime_genres.loc[df_anime_genres.shape[0]] = [row[1]['mal_id'], genre['genre_id']] # add the genre to the dataframe
            if(genre['genre_id'] not in df_genres_d['genre_id'].values): # if the genre is not in the dataframe
                df_genres_d.loc[df_genres_d.shape[0]] = [genre['genre_id'], genre['genre_de']] # if the genre not already in add it to the dataframe
```


```python
df_anime_demographic.sort_values(by=['mal_id', 'demo_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id and demo_id
df_anime_demographic.index.name = "Index" # set the index name to "Index"
df_anime_demographic.to_csv(DATA_FOLDER + 'anime_demographics.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_demographic.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>demo_id</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>6</td>
      <td>27</td>
    </tr>
    <tr>
      <th>1</th>
      <td>8</td>
      <td>27</td>
    </tr>
    <tr>
      <th>2</th>
      <td>15</td>
      <td>27</td>
    </tr>
    <tr>
      <th>3</th>
      <td>16</td>
      <td>43</td>
    </tr>
    <tr>
      <th>4</th>
      <td>17</td>
      <td>27</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_demographic_d.sort_values(by=['demo_id'], inplace=True, ignore_index=True) # sort the dataframe by the demo_id
df_demographic_d.index.name = "Index" # set the index name to "Index"
df_demographic_d.to_csv(DATA_FOLDER + 'demographics_d.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_demographic_d.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>demo_id</th>
      <th>demo_de</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>15</td>
      <td>Kids</td>
    </tr>
    <tr>
      <th>1</th>
      <td>25</td>
      <td>Shoujo</td>
    </tr>
    <tr>
      <th>2</th>
      <td>27</td>
      <td>Shounen</td>
    </tr>
    <tr>
      <th>3</th>
      <td>42</td>
      <td>Seinen</td>
    </tr>
    <tr>
      <th>4</th>
      <td>43</td>
      <td>Josei</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_genres_d.sort_values(by=['genre_id'], inplace=True, ignore_index=True) # sort the dataframe by the genre_id
df_genres_d.index.name = "Index" # set the index name to "Index"
df_genres_d.to_csv(DATA_FOLDER + 'genres_d.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_genres_d.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>genre_id</th>
      <th>genre_de</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Action</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Adventure</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Racing</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>Comedy</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>Avant Garde</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_anime_genres.sort_values(by=['mal_id', 'genre_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id and by genre_id
df_anime_genres.index.name = "Index" # set the index name to "Index"
df_anime_genres.to_csv(DATA_FOLDER + 'anime_genres.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_genres.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>genre_id</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>24</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>29</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1</td>
      <td>50</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime media types and nsfw to csv file


```python
df_anime_media_type = df_anime[['mal_id', 'media_type', 'nsfw']].copy() # copy the dataframe columns to a new dataframe
df_anime_media_type.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_media_type.index.name = "Index" # set the index name to "Index"
df_anime_media_type.to_csv(DATA_FOLDER + 'media_type_d.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_media_type.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>media_type</th>
      <th>source</th>
      <th>nsfw</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>tv</td>
      <td>original</td>
      <td>white</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>movie</td>
      <td>original</td>
      <td>white</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>tv</td>
      <td>manga</td>
      <td>white</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>tv</td>
      <td>original</td>
      <td>white</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>tv</td>
      <td>manga</td>
      <td>white</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime sources to csv file


```python
df_anime_source = df_anime[['mal_id', 'source']].copy() # copy the dataframe columns to a new dataframe
df_anime_source.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_source.index.name = "Index" # set the index name to "Index"
df_anime_source.to_csv(DATA_FOLDER + 'anime_source.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_source.head()
```

### Save anime synopses to csv file


```python
df_synopsis_d = df_anime[['mal_id', 'synopsis']].copy() # copy the dataframe columns to a new dataframe
df_synopsis_d.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_synopsis_d.index.name = "Index" # set the index name to "Index"
df_synopsis_d.to_csv(DATA_FOLDER + 'synopsis_d.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_synopsis_d.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>synopsis</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Crime is timeless. By the year 2071, humanity ...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>Another day, another bounty—such is the life o...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>Vash the Stampede is the man with a $$60,000,0...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>Witches are individuals with special powers li...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>It is the dark century and the people are suff...</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime studios to csv file


```python
df_anime_studios = pd.DataFrame(columns=['mal_id', 'studio_id']) # create a dataframe with the columns
df_studios_d = pd.DataFrame(columns=['studio_id', 'studio_de']) # create a dataframe with the columns

cnt = 0
for row in df_anime.iterrows(): # iterate through the dataframe
    studios_str = row[1]['studios']
    if(pd.isna(genres_str)):
        continue
    studios_str = '{"studios": ' + studios_str.replace("id", 'studio_id').replace("name", 'studio_de') + '}'
    studio_d = ast.literal_eval(studios_str) # convert the string to a dictionary
    studios_d = pd.json_normalize(studio_d['studios']) # convert the dictionary to a dataframe
    
    df_studios_d = pd.concat([df_studios_d, studios_d], ignore_index=True).drop_duplicates() # concatenate the dataframes
    if(studios_d.empty): # if the dataframe is empty
        df_anime_studios.loc[df_anime_studios.shape[0]] = [int(row[1]['mal_id']), np.nan] # add the row to the dataframe and set it to NaN
        continue
    for studio in studio_d['studios']: # iterate through the studios
        df_anime_studios.loc[df_anime_studios.shape[0]] = [int(row[1]['mal_id']), int(studio['studio_id'])] # add the row to the dataframe    
```


```python
df_studios_d.sort_values(by=['studio_id'], inplace=True, ignore_index=True) # sort the dataframe by the genre_id
df_studios_d.index.name = "Index" # set the index name to "Index"
df_studios_d.to_csv(DATA_FOLDER + 'studios_d.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_studios_d.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>studio_id</th>
      <th>studio_de</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Pierrot</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Kyoto Animation</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Gonzo</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>Bones</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>Bee Train</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_anime_studios.sort_values(by=['mal_id', 'studio_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id and by genre_id
df_anime_studios.index.name = "Index" # set the index name to "Index"
df_anime_studios["mal_id"] = df_anime_studios["mal_id"].astype(np.int64) # convert the mal_id to an integer
df_anime_studios["studio_id"] = df_anime_studios["studio_id"].fillna(0).astype(np.int64) # convert the studio_id to an integer
df_anime_studios.to_csv(DATA_FOLDER + 'anime_studios.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_studios.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>studio_id</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>14</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>4</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>11</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>14</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>18</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime recommendations to csv file


```python
df_anime_recommendations = pd.DataFrame(columns=['mal_id', 'mal_id_rd', 'num_recommendations']) # create a dataframe with the columns
for row in df_anime.iterrows(): # iterate through the dataframe
    recommended_str = row[1]['recommendations']
    if(pd.isna(recommended_str)): # if the recommended_str is NaN
        continue
    recommended_str = '{"recommendations": ' + str(recommended_str) + '}'
    recom_d = ast.literal_eval(recommended_str) # convert the string to a dictionary
    for recommendation in recom_d['recommendations']: # iterate through the recommendations
        df_anime_recommendations.loc[df_anime_recommendations.shape[0]] = [row[1]['mal_id'], recommendation['node']['id'], recommendation['num_recommendations']] # add the row to the dataframe
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>mal_id_rd</th>
      <th>num_recommendations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>5114</td>
      <td>11061</td>
      <td>101</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5114</td>
      <td>16498</td>
      <td>39</td>
    </tr>
    <tr>
      <th>2</th>
      <td>5114</td>
      <td>1482</td>
      <td>23</td>
    </tr>
    <tr>
      <th>3</th>
      <td>5114</td>
      <td>9919</td>
      <td>17</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5114</td>
      <td>1575</td>
      <td>15</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_anime_recommendations.sort_values(by=['mal_id', 'num_recommendations'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id and by num_recommendations
df_anime_recommendations.index.name = "Index" # set the index name to "Index"
df_anime_recommendations.to_csv(DATA_FOLDER + 'anime_recommendations.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_recommendations.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>mal_id_rd</th>
      <th>num_recommendations</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>13601</td>
      <td>13</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>918</td>
      <td>14</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>2025</td>
      <td>16</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1</td>
      <td>4087</td>
      <td>18</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1</td>
      <td>2251</td>
      <td>30</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime relations to csv file


```python
df_anime_relations = pd.DataFrame(columns=['mal_id', 'mal_id_re', 'relation_type']) # create a dataframe with the columns
for row in df_anime.iterrows(): # iterate through the dataframe
    related_str = row[1]['related_anime']
    if(pd.isna(related_str)): # if the related_str is NaN
        continue
    related_d = ast.literal_eval(related_str) # convert the string to a dictionary
    for relation in related_d: # iterate through the relations
        df_anime_relations.loc[df_anime_relations.shape[0]] = [row[1]['mal_id'], relation['node']['id'], relation['relation_type']] # add the row to the dataframe
```


```python
df_anime_relations.sort_values(by=['mal_id', 'mal_id_re'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id and by mal_id_re
df_anime_relations.index.name = "Index" # set the index name to "Index"
df_anime_relations.to_csv(DATA_FOLDER + 'anime_relations.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_relations.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>mal_id_re</th>
      <th>relation_type</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>5</td>
      <td>side_story</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>4037</td>
      <td>summary</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>17205</td>
      <td>side_story</td>
    </tr>
    <tr>
      <th>3</th>
      <td>5</td>
      <td>1</td>
      <td>parent_story</td>
    </tr>
    <tr>
      <th>4</th>
      <td>6</td>
      <td>4106</td>
      <td>side_story</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime cover images to csv file (urls)


```python
df_anime_covers = df_anime[['mal_id', 'main_picture.medium', 'main_picture.large']].copy() # copy the dataframe columns to a new dataframe #'pictures' 
df_anime_covers.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_covers.rename(columns={"main_picture.medium": "main_picture_medium"}, inplace=True) # rename the column
df_anime_covers.rename(columns={"main_picture.large": "main_picture_large"}, inplace=True)
df_anime_covers.index.name = "Index" # set the index name to "Index"
df_anime_covers.to_csv(DATA_FOLDER + 'anime_covers.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_covers.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>main_picture.medium</th>
      <th>main_picture.large</th>
      <th>pictures</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>https://api-cdn.myanimelist.net/images/anime/4...</td>
      <td>https://api-cdn.myanimelist.net/images/anime/4...</td>
      <td>[{'medium': 'https://api-cdn.myanimelist.net/i...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>https://api-cdn.myanimelist.net/images/anime/1...</td>
      <td>https://api-cdn.myanimelist.net/images/anime/1...</td>
      <td>[{'medium': 'https://api-cdn.myanimelist.net/i...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>https://api-cdn.myanimelist.net/images/anime/7...</td>
      <td>https://api-cdn.myanimelist.net/images/anime/7...</td>
      <td>[{'medium': 'https://api-cdn.myanimelist.net/i...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>https://api-cdn.myanimelist.net/images/anime/1...</td>
      <td>https://api-cdn.myanimelist.net/images/anime/1...</td>
      <td>[{'medium': 'https://api-cdn.myanimelist.net/i...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>https://api-cdn.myanimelist.net/images/anime/7...</td>
      <td>https://api-cdn.myanimelist.net/images/anime/7...</td>
      <td>[{'medium': 'https://api-cdn.myanimelist.net/i...</td>
    </tr>
  </tbody>
</table>
</div>



### Save anime statistics to csv file


```python
df_anime_stats = df_anime[['mal_id', 'num_episodes', 'average_episode_duration', 
                           'statistics.status.watching', 'statistics.status.completed', 
                           'statistics.status.on_hold', 'statistics.status.dropped', 
                           'statistics.status.plan_to_watch', 'statistics.num_list_users']].copy() # copy the dataframe columns to a new dataframe
df_anime_stats.sort_values(by=['mal_id'], inplace=True, ignore_index=True) # sort the dataframe by the mal_id
df_anime_stats.rename(columns={"statistics.status.watching": "num_watching"}, inplace=True) # rename the columns
df_anime_stats.rename(columns={"statistics.status.completed": "num_completed"}, inplace=True) 
df_anime_stats.rename(columns={"statistics.status.on_hold": "num_on_hold"}, inplace=True)
df_anime_stats.rename(columns={"statistics.status.dropped": "num_dropped"}, inplace=True)
df_anime_stats.rename(columns={"statistics.status.plan_to_watch": "num_plan_to_watch"}, inplace=True)
df_anime_stats.rename(columns={"statistics.num_list_users": "num_list_users"}, inplace=True)
df_anime_stats.index.name = "Index" # set the index name to "Index"
df_anime_stats.to_csv(DATA_FOLDER + 'anime_stats.csv', sep=';', encoding='utf-8') # save the dataframe to a csv file
df_anime_stats.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mal_id</th>
      <th>num_episodes</th>
      <th>average_episode_duration</th>
      <th>statistics.status.watching</th>
      <th>statistics.status.completed</th>
      <th>statistics.status.on_hold</th>
      <th>statistics.status.dropped</th>
      <th>statistics.status.plan_to_watch</th>
      <th>statistics.num_list_users</th>
    </tr>
    <tr>
      <th>Index</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>26</td>
      <td>1440</td>
      <td>151257</td>
      <td>921226</td>
      <td>91835</td>
      <td>35042</td>
      <td>417977</td>
      <td>1617337</td>
    </tr>
    <tr>
      <th>1</th>
      <td>5</td>
      <td>1</td>
      <td>6911</td>
      <td>5606</td>
      <td>249064</td>
      <td>2472</td>
      <td>974</td>
      <td>76075</td>
      <td>334191</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>26</td>
      <td>1480</td>
      <td>35889</td>
      <td>393056</td>
      <td>29722</td>
      <td>16350</td>
      <td>184521</td>
      <td>659538</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7</td>
      <td>26</td>
      <td>1500</td>
      <td>4914</td>
      <td>49236</td>
      <td>5571</td>
      <td>5785</td>
      <td>40071</td>
      <td>105577</td>
    </tr>
    <tr>
      <th>4</th>
      <td>8</td>
      <td>52</td>
      <td>1380</td>
      <td>709</td>
      <td>7754</td>
      <td>801</td>
      <td>1178</td>
      <td>3861</td>
      <td>14303</td>
    </tr>
  </tbody>
</table>
</div>


