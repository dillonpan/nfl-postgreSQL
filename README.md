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
headers = afc_overall[0]
standings_header = []
for h in headers.find_all('th'):
    standings_header.append(h.get('data-stat'))
print(standings_header)
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

# Creating the PostgreSQL Database, Schema, and Table
Now that we have the first table, let's create the database + standings table before we move on to the other two data sets. In Python, there's a PostgreSQL database adapter named Psycopg. The current package is psycopg2, which we imported in the beginning. We will also be able to double check our creations using pgAdmin4, a management application that you can get while downloading PostgreSQL.

Note: For personal security, I have replaced my actual username & password with the following: [username] & [password]

Before we can execute queries, we have to connect to the server by creating a connect object. Afterwards, we create a cursor on the connection to execute queries. Any changes to the database requires a commit statement, but we can just set autocommit to True as a shortcut.
```
# Connect to PostgresSQL server, in general
conn = psycopg2.connect(user=[username], password=[password])
conn.autocommit = True
cur = conn.cursor()

# Create the database
cur.execute('CREATE DATABASE nfl_2019;')

# Close the cursor & connection
cur.close()
conn.close()

# Re-connect to server directly in to the newly created database
conn = psycopg2.connect(dbname="nfl_2019", user=[username], password=[password])
conn.autocommit = True
cur = conn.cursor()
```

How PostgreSQL works is that there is data within -> tables -> schemas -> databases. Once a database is created, a general default "public" schema is created as well. However, let's create a new schema for our project to place tables within. Second, remember how we placed the standings headers in a list called "standings_header"? It looked like this:  

['team', 'wins', 'losses', 'ties', 'win_loss_perc', 'points', 'points_opp', 'points_diff', 'mov', 'sos_total', 'srs_total', 'srs_offense', 'srs_defense']

We can now use the indexes of this list and some string formatting to create and label the data types of each column for our standings table under the nfl schema:
```
cur.execute('''
CREATE SCHEMA nfl;

CREATE TABLE nfl.standings(
    {} TEXT NOT NULL PRIMARY KEY,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} FLOAT NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} FLOAT NOT NULL,
    {} FLOAT NOT NULL,
    {} FLOAT NOT NULL,
    {} FLOAT NOT NULL,
    {} FLOAT NOT NULL
);'''.format(standings_header[0], standings_header[1], standings_header[2], standings_header[3], standings_header[4],
             standings_header[5],
             standings_header[6], standings_header[7], standings_header[8], standings_header[9], standings_header[10],
             standings_header[11],
             standings_header[12]))
```

# Inserting Data to nfl.standings Table
We can now insert the values within both AFC and NFC standings lists into the newly created table. Remember how each team and their respective data takes up 13 indexes? We can take that in to consideration as well. Another thing about the function below is that we are inserting smaller lists of team data as values. The "%s" are placeholders for values within the list so we dont need to go through each cell of the team list. When we created the tables, we listed what kind of data each column was so it will be read and converted through PostgreSQL:
```
# create function to import data to team table
def import_standings_data(standings_list):
    first = 0
    last = 13
    while last <= len(standings_list):
        team = standings_list[first:last]
        cur.execute('''
            INSERT INTO nfl.standings
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
            ''', team)
        first += 13
        last += 13


# use the function for both conferences
import_standings_data(afc_standings)
import_standings_data(nfc_standings)
```

Normally, you could use a print(cursor.fetchall()) line after executing a Select statement to print it but the staement will come out as a single line and the columns you chose won't appear either. The below function will print the Select statements as individual lines and will also include the selected columns names as well:

```
def printsql():
    try:
        column_names = [desc[0] for desc in cur.description]
        print(column_names)
        for line in cur.fetchall():
            print(line)
        print()
    except:
        print('no printable SQL execution')
```

Now we have created the database, schema, standings table and inserting the values to the table. Let's do a test Select statement and print it out:
```
cur.execute('''
    SELECT *
    FROM nfl.standings
    ORDER BY team
    LIMIT 3
''')

printsql()
```
['team', 'wins', 'losses', 'ties', 'win_loss_perc', 'points', 'points_opp', 'points_diff', 'mov', 'sos_total', 'srs_total', 'srs_offense', 'srs_defense']  
('Arizona Cardinals', 5, 10, 1, 0.344, 361, 442, -81, -5.1, 1.8, -3.2, -0.3, -2.9)  
('Atlanta Falcons', 7, 9, 0, 0.438, 381, 399, -18, -1.1, 1.1, -0.1, 0.3, -0.4)  
('Baltimore Ravens', 14, 2, 0, 0.875, 531, 282, 249, 15.6, 0.1, 15.6, 11.0, 4.7)  

# Pulling Data from a CSV File and Creating nfl.offense Table
So the next data set we will import in to PostgreSQL table is the offensive stats of all NFL teams during the 2019 regular season. This could have also been pulled from the website via web scrapping, however I decided to export the table in to a CSV to show an alternative method. This process is much simpler than web scraping only because we don't have unnecesary information like we did going through the entire html webpage.
The first thing we need to do is use the pandas package within Python to read the file. Afterwards, just like the standings data, we can make a list for the offense column headers:  
Note: I've replaced the destination folder of my file with [file location]
```
o_data = pandas.read_csv('[file location]OffenseStats.csv')

o_headers = list(o_data.columns)
print(o_headers)
```

['team', 't_points', 'fumbles_lost', 'pass_cpt', 'pass_att', 'pass_yrds', 'pass_tds', 'ints', 'rush_att', 'rush_yrds', 'rush_tds']

Since we are still connected to the database, we don't need to reconnect to it. Just like we did previously wth the standings, we can use string formatting and the headers list to create the offense table under the nfl schema. We also created a foreign key to link with the nfl.standings table:
```
cur.execute('''
CREATE TABLE nfl.offense(
    {} TEXT NOT NULL PRIMARY KEY REFERENCES nfl.standings(team),
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL
);'''.format(o_headers[0], o_headers[1], o_headers[2], o_headers[3], o_headers[4], o_headers[5],
             o_headers[6], o_headers[7], o_headers[8], o_headers[9], o_headers[10]))
```

# Inserting Data in to nfl.offense
The issue with linking pandas dataframes to PostgreSQL is that numbers through pandas are generally NumPy data types, which PostgreSQL does not accept. To solve this issue, we can just turn the NumPy types in to string types and let PostgreSQL readjust them to the data type we labeled for each column. The function below takes in each row one by one in the pandas dataframe and inerts the data in to your table of choice, in ths case nf.offense. We also set it up so different dataframes with different amounts of columns can still insert data:

```
def insert_values(data, table_n):
    for x in range(len(data)):
        row = data.iloc[x, :].tolist()
        str_row = [str(x) for x in row]
        cur.execute('''
            INSERT INTO {}
            VALUES ({});
            '''.format(table_n, ('%s,' * len(data.columns)).rstrip(',')), str_row)
```

Now we can use the function above to insert our o_data in to nfl.offense. Afterwards, let's do a test Select statement a a check:
```
insert_values(o_data, 'nfl.offense')

cur.execute('''
    SELECT *
    FROM nfl.offense
''')

printsql()
```
['team', 't_points', 'fumbles_lost', 'pass_cpt', 'pass_att', 'pass_yrds', 'pass_tds', 'ints', 'rush_att', 'rush_yrds', 'rush_tds']  
('Arizona Cardinals', 361, 6, 355, 554, 3477, 20, 12, 396, 1990, 18)  
('Atlanta Falcons', 381, 10, 459, 684, 4714, 29, 15, 362, 1361, 10)  
('Baltimore Ravens', 531, 7, 289, 440, 3225, 37, 8, 596, 3296, 21)  

# Pulling and Cleaning Data to Create nfl.players Table
Just like the offensive statistics, the data on players are in a CSV file. Thus, we will use pandas to open it and create a dataframe:
```
p_data = pandas.read_csv('PlayerStats.csv')

p_headers = p_data.columns.tolist()
print(p_headers)
```  
['name', 'team', 'position', 'age', 'games', 'g_started', 'pass_com', 'pass_att', 'pass_yrds', 'pass_tds', 'pass_ints', 'rush_att', 'rush_yrds', 'rush_tds', 'rec_tgts', 'rec_catch', 'rec_yrds', 'rec_tds', 'fumbles', 'fum_lost', 'fan_pts']


