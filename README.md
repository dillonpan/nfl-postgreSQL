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

# Pulling/Cleaning Data and Building nfl.players Table
Just like the offensive statistics, the data on players are in a CSV file. Thus, we will use pandas to open it and create a dataframe:
```
p_data = pandas.read_csv('PlayerStats.csv')

p_headers = p_data.columns.tolist()
print(p_headers)
```  
['name', 'team', 'position', 'age', 'games', 'g_started', 'pass_com', 'pass_att', 'pass_yrds', 'pass_tds', 'pass_ints', 'rush_att', 'rush_yrds', 'rush_tds', 'rec_tgts', 'rec_catch', 'rec_yrds', 'rec_tds', 'fumbles', 'fum_lost', 'fan_pts']

Looking in to the CSV file itself, we've run in to an issue. For the players, the teams listed arent the full names but the team abbreviations. Example of CSV column for "team":

team
------  
CAR  
BAL  
TEN  
GNB  
DAL  

One way to replace the abbrivations with their full names is to create a map and apply it to the "team" column:
```
team_dict = {
    "ARI": "Arizona Cardinals",
    "ATL": "Atlanta Falcons",
    "BAL": "Baltimore Ravens",
    "BUF": "Buffalo Bills",
    "CAR": "Carolina Panthers",
    "CHI": "Chicago Bears",
    "CIN": "Cincinnati Bengals",
    "CLE": "Cleveland Browns",
    "DAL": "Dallas Cowboys",
    "DEN": "Denver Broncos",
    "DET": "Detroit Lions",
    "GNB": "Green Bay Packers",
    "HOU": "Houston Texans",
    "IND": "Indianapolis Colts",
    "JAX": "Jacksonville Jaguars",
    "KAN": "Kansas City Chiefs",
    "LAC": "Los Angeles Chargers",
    "LAR": "Los Angeles Rams",
    "MIA": "Miami Dolphins",
    "MIN": "Minnesota Vikings",
    "NOR": "New Orleans Saints",
    "NWE": "New England Patriots",
    "NYG": "New York Giants",
    "NYJ": "New York Jets",
    "OAK": "Oakland Raiders",
    "PHI": "Philadelphia Eagles",
    "PIT": "Pittsburgh Steelers",
    "SEA": "Seattle Seahawks",
    "SFO": "San Francisco 49ers",
    "TAM": "Tampa Bay Buccaneers",
    "TEN": "Tennessee Titans",
    "WAS": "Washington Redskins"
}

p_data['team'] = p_data['team'].map(team_dict)
```

Let's take a look at the value counts for the "team" column now to ensure our map worked. Let's also see how many total players are in our file:
```
print(len(p_data))

print(p_data['team'].value_counts(dropna=False))
```
450
Detroit Lions           17  
NaN                     16  
New York Giants         16  
Atlanta Falcons         16  
Cleveland Browns        15  

Looks like our map generally worked except we have some NaN values, which is weird. Upon further review, it seems the NaN values were for players who played with more than a single team during the 2019 season. We prefer players who played on a single team so let's drop these 16 players since theres still 434 players we can use:
```
p_data = p_data.dropna(subset=['team'], axis=0)  # drop the entire row, default axis =0
```

Just like with the other tables, let's create the table and insert the values for nfl.players. Just like with nfl.offense, we will create a foreign key for nfl.players that will connect the team name to the team name column in nfl.standings. Afterwards, we can create a test Select statement. This time we'll use a count statement to show the table includes the remaining 434 players:
```
cur.execute('''
CREATE TABLE nfl.players(
    {} TEXT NOT NULL PRIMARY KEY,
    {} TEXT NOT NULL REFERENCES nfl.standings(team),
    {} TEXT NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
    {} INTEGER NOT NULL,
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
);'''.format(p_headers[0], p_headers[1], p_headers[2], p_headers[3], p_headers[4], p_headers[5],
             p_headers[6], p_headers[7], p_headers[8], p_headers[9], p_headers[10], p_headers[11],
             p_headers[12], p_headers[13], p_headers[14], p_headers[15], p_headers[16],
             p_headers[17], p_headers[18], p_headers[19], p_headers[20]))

insert_values(p_data, 'nfl.players')

cur.execute('''
    SELECT COUNT(*)
    FROM nfl.players
''')

printsql()
```

['count']
(434,)

Perfect! Now that all 3 tables have been created and contain their respective data, we can run some SQL statements to answer some questions we may have had about the season:

Q1. List the total team passing tds, the players who caught at least 1 td, and the number of catching tds they had this year. Order the query by who caught the most tds first:
```
cur.execute('''
    SELECT standings.team, offense.pass_tds, players.name, players.rec_tds
    FROM nfl.standings
    JOIN nfl.offense ON offense.team = standings.team
    JOIN nfl.players ON players.team = offense.team
    WHERE standings.team = 'San Francisco 49ers' AND players.rec_tds >= 1
    ORDER BY players.rec_tds DESC
''')

printsql()
```
['team', 'pass_tds', 'name', 'rec_tds']  
('San Francisco 49ers', 28, 'Kendrick Bourne', 5)  
('San Francisco 49ers', 28, 'George Kittle', 5)  
('San Francisco 49ers', 28, 'Deebo Samuel', 3)  
('San Francisco 49ers', 28, 'Raheem Mostert', 2)  
('San Francisco 49ers', 28, 'Dante Pettis', 2)  
('San Francisco 49ers', 28, 'Ross Dwelley', 2)  
('San Francisco 49ers', 28, 'Richie James', 1)  
('San Francisco 49ers', 28, 'Tevin Coleman', 1)  
('San Francisco 49ers', 28, 'Matt Breida', 1)  
('San Francisco 49ers', 28, 'Jeff Wilson', 1)  
('San Francisco 49ers', 28, 'Kyle Juszczyk', 1)  
('San Francisco 49ers', 28, 'Marquise Goodwin', 1)  

Q2. For each team, list the players with at least 1 reception, their rec_yrds, and how much percentage of the teams receiving yrds they contributed to:
```
cur.execute('''
    SELECT players.team, players.name, players.rec_yrds, ROUND((100.0*players.rec_yrds)/offense.pass_yrds, 2)AS "percentage"
    FROM nfl.players
    JOIN nfl.offense ON offense.team = players.team
    WHERE players.rec_catch > 0
    GROUP BY players.team, players.name, offense.pass_yrds
    ORDER BY players.team, percentage DESC
''')

printsql()
```

['team', 'name', 'rec_yrds', 'percentage']  
('Arizona Cardinals', 'Larry Fitzgerald', 804, Decimal('23.12'))  
('Arizona Cardinals', 'Christian Kirk', 709, Decimal('20.39'))  
('Arizona Cardinals', 'David Johnson', 370, Decimal('10.64'))  
('Arizona Cardinals', 'Damiere Byrd', 359, Decimal('10.32'))  
('Arizona Cardinals', 'Charles Clay', 237, Decimal('6.82'))  
('Arizona Cardinals', 'Maxx Williams', 202, Decimal('5.81'))  
('Arizona Cardinals', 'Andy Isabella', 189, Decimal('5.44'))  
('Arizona Cardinals', 'KeeSean Johnson', 187, Decimal('5.38'))  
('Arizona Cardinals', 'Chase Edmonds', 105, Decimal('3.02'))  
('Arizona Cardinals', 'Trent Sherfield', 80, Decimal('2.30'))  
('Atlanta Falcons', 'Julio Jones', 1394, Decimal('29.57'))  
('Atlanta Falcons', 'Calvin Ridley', 866, Decimal('18.37'))  
('Atlanta Falcons', 'Austin Hooper', 787, Decimal('16.69'))  
etc  
etc  

Q3. Which teams are in the top 10 in terms of offensive yardage and have an average team age younger than that of an average NFL player age? What are these team's records?
```
cur.execute('''
    SELECT standings.team, standings.wins || '-' || standings.losses AS record,
    (offense.pass_yrds + offense.rush_yrds) AS total_yrds, ROUND(AVG(players.age), 2)
    FROM nfl.standings
    JOIN nfl.offense ON offense.team = standings.team
    JOIN nfl.players ON players.team = offense.team
    WHERE standings.team IN (SELECT team
                             FROM nfl.offense
                             ORDER BY (offense.pass_yrds + offense.rush_yrds) DESC
                             LIMIT 10)
    GROUP BY standings.team, record, total_yrds
    HAVING AVG(players.age) <  (SELECT AVG(age)
                                FROM nfl.players)
    ORDER BY total_yrds DESC
''')

printsql()
```

['team', 'record', 'total_yrds', 'round']  
('Baltimore Ravens', '14-2', 6521, Decimal('25.17'))  
('Tampa Bay Buccaneers', '7-9', 6366, Decimal('24.64'))  
('San Francisco 49ers', '13-3', 6097, Decimal('25.46'))  
('Los Angeles Rams', '9-7', 5998, Decimal('25.20'))  
('Seattle Seahawks', '11-5', 5991, Decimal('26.07'))  

Q4. List the teams where the lead running back has more than twice the amount of carries than the main backup (2nd highest in carries), ordered by teams. For fun, let's also check out the fantasy points these players had (under standard league settings).)This one is a little bit difficult in a single query so I will split this in to two then show a combined query:

First method: We're going to create a view that includes two things, each team and the 2nd highest RB rush attempts. We're going to use a window function and partitioning. Unlike Group By, Partition By will keep all rows and compare running backs within each team and rank them by descending rush attempts via the rank() function. Afterwards, the second query will compare each teams highest rusher to their respective team from the view we built first.
```
cur.execute('''
    CREATE VIEW rb_twos AS(
        SELECT team, rush_att
        FROM (SELECT team, name, rush_att, rank() OVER (PARTITION BY team ORDER BY rush_att DESC)
              FROM nfl.players
              WHERE position = 'RB') AS test
        WHERE rank = 2
    );
    
    (SELECT team, name
     FROM nfl.players
     WHERE position = 'RB'
     GROUP BY team, name
     HAVING MAX(rush_att) > 2*(SELECT rush_att
                               FROM rb_twos
                               WHERE rb_twos.team = players.team)
     ORDER BY team
    )
''')

printsql()
```
Here's the second method in which we combined them in to a single query with a subquery. Note that we needed to create an alias 'a' so players isn't used twice.:
```
cur.execute('''
    SELECT a.team, a.name
    FROM nfl.players a
    WHERE position = 'RB'
    GROUP BY a.team, a.name
    HAVING MAX(a.rush_att) > 2*(SELECT rush_att
                                FROM (SELECT team, name, rush_att, rank() OVER (PARTITION BY team ORDER BY rush_att DESC)
                                    FROM nfl.players
                                    WHERE position = 'RB') AS test
                                WHERE rank = 2 AND team = a.team)
    ORDER BY a.team
''')

printsql()
```
['team', 'name', 'fan_pts']  
('Atlanta Falcons', 'Devonta Freeman', 139)  
('Carolina Panthers', 'Christian McCaffrey', 355)  
('Chicago Bears', 'David Montgomery', 145)  
('Cincinnati Bengals', 'Joe Mixon', 190)  
('Cleveland Browns', 'Nick Chubb', 219)  
('Dallas Cowboys', 'Ezekiel Elliott', 258)  
('Green Bay Packers', 'Aaron Jones', 266)  
('Houston Texans', 'Carlos Hyde', 143)  
('Indianapolis Colts', 'Marlon Mack', 167)  
('Jacksonville Jaguars', 'Leonard Fournette', 183)  
('Los Angeles Rams', 'Todd Gurley', 188)  
('Minnesota Vikings', 'Dalvin Cook', 239)  
('New England Patriots', 'Sony Michel', 141)  
('New York Giants', 'Saquon Barkley', 192)  
('New York Jets', "Le'Veon Bell", 149)  
('Oakland Raiders', 'Josh Jacobs', 172)  
('Seattle Seahawks', 'Chris Carson', 196)  
('Tennessee Titans', 'Derrick Henry', 277)  
('Washington Redskins', 'Adrian Peterson', 130)  

To further this project, we could always import the team/individual player defensive stats and perform other types of analysis. But for now, this is just an example on how to gather data and use it to develop your own relational database.
