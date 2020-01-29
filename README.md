# nfl-postgreSQL
Project gathering and loading 2019 NFL regular season data into Python via web scraping/Excel sheets and various Python packages (requests, BeautifulSoup, pandas). Afterwards, take the data and create a relational database to execute various SQL queries through Python’s PostgreSQL package (psycop2)

# Project Details:
For this project we will create a new relational database connecting three tables of the following 2019 NFL rgular season data:
1. Team Standings/ various team statistics (Web scraped, [https://www.pro-football-reference.com/years/2019/])
2. Detailed statistics on team offenses (CSV sheet)
3. Offensive statistics of the top 450 players, based on default fantasy football scoring (CSV sheet)

To show that we can take data from multiple sources and combine them successfully, I retrieved data in two forms from the same source. All the data in this project came from [Pro Football Reference](https://www.pro-football-reference.com/) but we used web scraping only for the standings while the other two were exported to CSV docs.

# Scraping and Importing Standings to Python

We can use the package "requests" to pull the website and then take the websites content. Since the website is html, we can use the package "BeautifulSoup", which allows us to parse and retrieve the sections/data we need specifically off of the html webpage. psycopg2 & pandas will be used later.
```
import requests
from bs4 import BeautifulSoup
import psycopg2
import pandas

response = requests.get('https://www.pro-football-reference.com/years/2019/')
content = response.content
parser = BeautifulSoup(content, 'html.parser')
```
If we take a look in the webpage (https://www.pro-football-reference.com/years/2019/) itself, we can see that the standings are split in to AFC and NFC conferences and each have their own table. To check out the webpage and see what to parse out, we can inspect the section we're interested in. Let's inspect the first team just to see where the code is:
![image](https://user-images.githubusercontent.com/57373723/73312943-cad93880-41de-11ea-91a0-825d81eb2bd0.png)

If we scroll a little bit up on the html code, we can find the line which generally emcompasses the entire table:
![image](https://user-images.githubusercontent.com/57373723/73313063-0ffd6a80-41df-11ea-96a4-5339f530ffa7.png)

Now that we have the section we want, we can pull just that section from the entire html page based on the property and id of that html section. We can also specifically find and print the rows within that section afterwards for an overview look:
```
afc = parser.find('div', id='all_AFC')

afc_overall = afc.find_all('tr')

for section in afc_overall:
    print(section)
    print()
```
Note: "afc_overall" becomes a list type variable which holds each row in an index.

It's a lot of rows so I won't paste it here but in general it's lines starting with "<tr>" and ending with "</tr>". Once you have run the code, you'll notice theres about three kinds of rows which you can see on the webpage:
![image](https://user-images.githubusercontent.com/57373723/73314015-974bdd80-41e1-11ea-8a02-fdc776ffea15.png)

After running the above, youll notice the first printed row (starting with "<tr>" and ending with "</tr>") is this massive block of code representing the table column headers. The second type of row printed are the single rows only listing the division of the team. Lastly, theres the rows representing team data itself.

We're going to start pulling the data out of the section by getting the column headers, which will be used as the column headers in our relational database later. Just like how this entire section was under the ID 'all_AFC', each row has their own html attributes. Taking a look at the printed rows, we can see the attribute we could use as column headers is the "data-stat" attribute:
![image](https://user-images.githubusercontent.com/57373723/73314597-21487600-41e3-11ea-8b5a-9b9a880d2070.png)

You can see in the image above that each cell within the header row starts with "<th>" and ends with "</th>" (one of html codes for a cell). The following code will first pull the header row out of the afc_overall list of rows then run through each cell within the row to pull the data-stat attribute and append them to a list:
```
# find the team table names
headers = afc_overall[0]
team_standings = []
for h in headers.find_all('th'):
    team_standings.append(h.get('data-stat'))
print(team_standings)
```
['team', 'wins', 'losses', 'ties', 'win_loss_perc', 'points', 'points_opp', 'points_diff', 'mov', 'sos_total', 'srs_total', 'srs_offense', 'srs_defense']

Now that the future column headers for our standings table is done, let's pull the teams and their standings information. The following function will do just that. We're making it a function because we still need to pull the data for the NFC standings. Since both tables hold the same type of data, we can reuse this function. 

```
def pull_standings(conference):
    standings_list = []
    for team in conference[1:]:
        # below to filter out the division name rows
        if len(team) > 1:
            for stat in team:
                # some team names have an asterisk or plus sign so we will remove them
                stat = stat.text.replace('*', '').replace('+', '')
                standings_list.append(stat)
    return standings_list
```
```
afc_standings = pull_standings(afc_overall)
print(afc_standings)
```
['New England Patriots', '12', '4', '0', '.750', '420', '225', '195', '12.2', '-1.8', '10.4', '2.8', '7.6', 'Buffalo Bills', '10', '6', etc etc etc....

Let's go to the NFC side now. We can inspect the webpage as we previously did for the AFC side to find it's pretty much identical, just a slight id change from "all_AFC" to "all_NFC". We also don't need to redo the headers as we will merge both conferences under the same table and headers.
```
nfc = parser.find('div', id='all_NFC')
nfc_overall = nfc.find_all('tr')
nfc_standings = pull_standings(nfc_overall)
print(nfc_standings)
```
['Philadelphia Eagles', '9', '7', '0', '.563', '385', '354', '31', '1.9', '-1.7', '0.3', '0.7', '-0.4', 'Dallas Cowboys', '8', '8', etc etc etc...

We now have two long lists of names, standings, and stats of teams. We'll adjust both lists later on.

