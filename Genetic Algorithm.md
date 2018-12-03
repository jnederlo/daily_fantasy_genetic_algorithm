# Building a Genetic Algorithm in Python for Daily Fantasy Sports

Today I will show you how to use a Genetic Algorithm (GA) to create lineups for daily fantasy sports! A GA is inspired by the theory of evolution through the process of natural selection, and can be used to generate high-quality solutions to optimization problems, like building lineups. To oversimplify it, a GA works by starting with a group of randomly generated solutions - a population. The individuals within this population will reproduce and pass on some of their traits to their offspring. Sometimes these traits mutate and the mutation gets passed down. Over time the more desirable individuals will reproduce more, meaning their is a higher likelihood that their traits get passed down. 

For our example, think of a lineup as an individual, and randomly generated lineups as the population. We will mate lineups together to create new lineups, sometimes injecting random players to mimic a mutation. Over time, the better players will be passed on to more of our lineups letting us create better combinations of lineups overall. I've made it easy in that the end result of what we build will produce an exportable CSV file that can be uploaded directly into DraftKings, and the code is available to download from my Github account with a link at the bottom of the article. I've built this example for daily fantasy hockey as that is what I know best, but the same concepts and code could be used for other sports with minimal effort.


## Caution
**All of this might sound good on the surface, and I think there is some real value here, but it is really important to note that you can't expect to win daily fantasy sports contests just copying what I have done. There is a lot that goes into a successful daily fantasy player, most of which I don't cover here. In future posts I plan to discuss other methods that can be used, but I think this is a really good first step to take when learning what it takes to be successful.**

****

Now that we have that out of the way, we can begin. Start by logging into your DraftKings account, and download the available roster. Move the dowloaded file to the same folder/directory that contains the python code (the stuff we are about to write!). I use the DraftKings roster for a couple of reasons:

- It includes player ID's that are required to upload back to the site.
- It includes the player salary and position information.
- It includes average player fantasy points for the current season.

The last point is especially important for us. We will need some way to rank lineups, and to keep things simple I have decided to use a players average fantasy points. In practice it will be up to you to replace the average points with your own projections, remember, Garbage In = Garbage Out. You will also need to edit the roster to remove any players not playing, or ones you just don't like. But for now this is what you will use. 

We will write everything in Python, so it is necessary for you to have it installed on your computer. If you don't know if Python is installed on your computer or you don't know how to install it you can find out how on a [windows
](https://ehmatthes.github.io/pcc/chapter_01/windows_setup.html) or a [mac](https://ehmatthes.github.io/pcc/chapter_01/osx_setup.html) machine by clicking the correct link. I have written the code so that it works for python 2 or 3, so either one should work. Now for the code. 

### Step 1: Import packages and define the class
We will import a few necessary packages from the python standard library. Next we define a class which holds the number of lineups we want to create, the duration (in seconds) that we want the program to run, and a few empty lists corresponding to positions. A class isn't necessary but it is a convenient way to store our variables, and provides a good base to build additional features on, or to add more sports,

>**Note:**  As our algorithm runs we will continually keep track of our top 150 lineups because that is the max number of lineups you can enter in any single contest on DraftKings. Hence we initialize an empty list called "*top_150*".


```
import csv
import time
import random
import copy
  
class GeneticNHL(object):

	def __init__(self, num_lineups, duration=60):
		self.num_lineups = num_lineups
		self.duration = duration
		self.goalies = []
		self.centers = []
		self.wingers = []
		self.defencemen = []
		self.utils = []
		self.top_150 = []
```


### Step 2: Define a method to load up the roster
Now we can create a method that will load up the CSV file from DraftKings and fill all of those empty position lists we just created. We separate out the players into their individual positions because it makes it much easier to select them later on.


```
def load_roster(self):

	with open("./DKSalaries.csv") as f:
		reader = csv.reader(f)
	
		#skip past the instructions and header in file
		for _ in range(8):
			next(reader)

		#grab the the player information for each player
		for row in reader:
			player = {}
			player['name'] = row[11]
			player['position'] = row[14][0]
			player['salary'] = int(row[15])
			player['teamAbbrev'] = row[17]
			player['avePoints'] = float(row[18])

		#append each player dictionary to their corresponding position list
		#only grab players who's average points are not 0 (i.e. they are active)
		if player['avePoints'] != 0:
			if player['position'] == 'G':
				self.goalies.append(player)
			elif player['position'] == 'C':
				self.centers.append(player)
			elif player['position'] == 'W':
				self.wingers.append(player)
			elif player['position'] == 'D':
				self.defencemen.append(player)

		#append the player to the utility position list if not a goalie
		if player['position'] != 'G':
			self.utils.append(player)
```


### Step 3: Define methods to generate a lineup and check if it's valid
Now that we have the list of players loaded into the program we can start making lineups. Here we define a method that will construct a single lineup, we will also define another method to check whether the constructed lineup is valid. Inside of a while loop we create a lineup by randomly selecting players for each position from our position arrays. Once we have the correct number of players at each position we send the lineup to the check_valid() method which will ensure that it is a valid lineup. If we have a valid lineup we exit the while loop and return that lineup, otherwise we construct a new lineup and repeat.

>**Note:** A valid NHL lineup for classic contests on DraftKings contains exactly 2 Centers, 3 Wingers, 2 Defensemen, 1 Goalie, and 1 Utility player. Valid lineups also fit under the $50,000 salary cap, don't have duplicate players, and utilize players from at least 3 different teams.


```
def generate_lineup(self):
	while True:
		#add the correct number of each position to a lineup
		lineup = []
		lineup.append(self.centers[random.randint(0, len(self.centers) - 1)])
		lineup.append(self.centers[random.randint(0, len(self.centers) - 1)])
		lineup.append(self.wingers[random.randint(0, len(self.wingers) - 1)])
		lineup.append(self.wingers[random.randint(0, len(self.wingers) - 1)])
		lineup.append(self.wingers[random.randint(0, len(self.wingers) - 1)])
		lineup.append(self.defencemen[random.randint(0, len(self.defencemen) - 1)])
		lineup.append(self.defencemen[random.randint(0, len(self.defencemen) - 1)])
		lineup.append(self.goalies[random.randint(0, len(self.goalies) - 1)])
		lineup.append(self.utils[random.randint(0, len(self.utils) - 1)])

		#Check if the lineup is valid (i.e. it satisfies some basic constraints)
		lineup = self.check_valid(lineup)
		if lineup:
			return lineup
```

```
def check_valid(self, lineup):
	#calculate the total projection of the lineup based on player averages
	projection = round(sum(player['avePoints'] for player in lineup), 2)
	
	#calculate the total salary used for the lineup
	salary = sum(player['salary'] for player in lineup)

	#get a list of the unique teams on the lineup
	teams = set([player['teamAbbrev'] for player in lineup])

	#remove duplicate players from lineup and count how many players
	num_players = len(set(player['name'] for player in lineup))

	#check the salary cap, at least 3 teams, and at least 9 unique players
	if salary < 50000 and len(teams) >= 3 and num_players == 9:
		#add the salary and the projection to the lineup of players and return the lineup
		lineup.extend((salary, projection))
		return lineup
	return  False
```

### Step 4: Define a method to mate two lineups together

We have a way to create valid lineups, the next step is start making lineups and combining (mating) them together. We are getting to the heart of what makes it a genetic algorithm.

To mate our lineups we define a new method that will take in two lineups to be mated together. We create position lists from the available players from both lineups, adding in a random player or two. For example, from each of the two lineups passed in we have two centers, we then add in another random center to give us five available centers to choose for our new lineup. We do this for each position. We then define a small helper function that will randomly grab the correct number of players from each position array to make a new lineup. As before, we will use a while loop to actually create the lineup, ensuring our mated lineup is valid before we return it.


```
def mate_lineups(self, lineup1, lineup2):

	#Create lists of all available players for each position from the two lineups plus a random player or 2
	centers = [lineup1[0], lineup1[1], lineup2[0], lineup2[1], self.centers[random.randint(0, len(self.centers) - 1)]]
	wingers = [lineup1[2], lineup1[3], lineup1[4], lineup2[2], lineup2[3], lineup2[4], self.wingers[random.randint(0, len(self.wingers) - 1)]]
	defencemen = [lineup1[5], lineup1[6], lineup2[5], lineup2[6], self.defencemen[random.randint(0, len(self.defencemen) - 1)]]
	goalies = [lineup1[7], lineup2[7], self.goalies[random.randint(0, len(self.goalies) - 1)], self.goalies[random.randint(0, len(self.goalies) - 1)]]
	utils = [lineup1[8], lineup2[8], self.utils[random.randint(0, len(self.utils) - 1)], self.utils[random.randint(0, len(self.utils) - 1)]]

	#Randomly grab num players from each position to fill out the new mated lineup
	def grab_players(players, num):
		avail_players = copy.deepcopy(players)
		selected_players = []
		while  len(selected_players) < num:
			i = random.randint(0, len(avail_players) -  1)
			selected_players.append(avail_players[i])
			del avail_players[i]
		return selected_players

	while True:
		#Create the new lineup by selecting players from each position list
		lineup = []
		lineup.extend(grab_players(centers, 2))
		lineup.extend(grab_players(wingers, 3))
		lineup.extend(grab_players(defencemen, 2))
		lineup.extend(grab_players(goalies, 1))
		lineup.extend(grab_players(utils, 1))
	
		#Check if the lineup is valid (i.e. it satisfies some basic constraints)
		lineup = self.check_valid(lineup)
		#If lineup is valid, return it, otherwise keep trying
		if lineup:
			return lineup
``` 


### Step 5: Define a method to execute the generation of and mating of lineups
Finally, we are ready to start making lineups, this is where you get to be creative. Define a new method called get_lineups(). This method will call our create_lineups() method and will decide on which two lineups to mate together.

We will start by generating 10 random lineups. We then rank the lineups by their projected score (the players average scores summed together) and add them to our top_150 lineups list. Next, we take our top three lineups from the 10 we just created and we mate them together to create three new offspring. As an extra step, and to make our algorithm more greedy, we mate each of the offspring with a randomly selected lineup from our top 150. As a final step, we add each of the original offspring, and the new offspring to our top 150 lineups.
> **Note:** Feel free to play around with this method. You could mate more lineups together, add new random lineups in, or mate lineups together as many times as you want. 


```
def get_lineups(self):
	#generate 10 new lineups
	new_lineups = [self.generate_lineup() for _ in range(10)]

	#sort the lineups by their predicted score
	new_lineups.sort(key=lambda x: x[-1], reverse=True)
	
	#Add the newly created lineups to the top_150 (they will be sorted and bottom ones removed later)
	self.top_150.extend(new_lineups)
	
	#Mate the top 3 lineups together
	offspring_1 = self.mate_lineups(new_lineups[0], new_lineups[1])
	offspring_2 = self.mate_lineups(new_lineups[0], new_lineups[2])
	offspring_3 = self.mate_lineups(new_lineups[1], new_lineups[2])
	
	#Mate the offspring with a randomly selected lineup from the top_150 and add to top_150
	#Adding this step makes the algorithm more greedy, and produces higher projections, but can be skipped
	self.top_150.append(self.mate_lineups(offspring_1, self.top_150[random.randint(0, len(self.top_150) - 1)]))
	self.top_150.append(self.mate_lineups(offspring_2, self.top_150[random.randint(0, len(self.top_150) - 1)]))
	self.top_150.append(self.mate_lineups(offspring_3, self.top_150[random.randint(0, len(self.top_150) - 1)]))

	#Add the original offspring to the top_150
	self.top_150.append(offspring_1)
	self.top_150.append(offspring_2)
	self.top_150.append(offspring_3)
```


### Step 6: Define a method to run the program
We have all of the ingredients to make our lineups, but now we have to actually run the algorithm. We will define a run() method that will continually call our get_lineups() method as many times as possible in the runtime duration we specify. For every iteration we will rank our top 150 lineups and slice the list to ensure that we are only including the top 150 lineups ranked by their projected points. 

> **Note:** I suggest a runtime of 60 seconds, which I have made the default - this should be enough time to let the algorithm find good combinations of lineups, but you can increase it or decrease it as you like, just pass in a different duration in seconds when you initialize the class.


```
def run(self):
	runtime = time.time() + self.duration
	while time.time() < runtime:
		self.get_lineups()
		self.top_150.sort(key=lambda x: x[-1], reverse=True)
		#We use 150 as the number here b/c drafkings only allows a max of 150 lineups in any one contest
		self.top_150 = self.top_150[:150]
```


### Step 7: Define a method to save our generated lineups
As a last step before we run the whole program, we want to save our created lineups. Here we define a save_file() method that will save two new CSV files. One file will include the total salary used and the projection of each created lineup, and the other file will only include the player name + id - this is the file you can upload directly to DraftKings. The first thing we do is trim all of the lineups to include only the player name + id, the salary used, and the total projection. We then remove any duplicate lineups, and create a version of the lineups for upload. We define a few headers and then write CSV files. The files will be created in the same folder/directory as the program.


```
def save_file(self):
	#trim the lineups to include only the player name + id, salary and projection
	lineups = [[player['name'] if isinstance(player, dict) else player for player in lineup] for lineup in self.top_150]

	#remove the duplicate lineups
	lineups = [lineups[i] for i in range(self.num_lineups - 1) if lineups[i] != lineups[i+1]]

	#remove the salary and projection so can upload lineups to DraftKings
	lineups_for_upload = [lineup[:9] for lineup in lineups]

	header = ['C', 'C', 'W', 'W', 'W', 'D', 'D', 'G', 'UTIL', 'Salary', 'Projection']
	header_for_upload = ['C', 'C', 'W', 'W', 'W', 'D', 'D', 'G', 'UTIL']

	with  open("./lineups.csv", 'w') as f:
		writer = csv.writer(f)
		writer.writerow(header)
		writer.writerows(lineups)

	with  open("./lineups_for_upload.csv", 'w') as f:
		writer = csv.writer(f)
		writer.writerow(header_for_upload)
		writer.writerows(lineups_for_upload)
```


### Step 8: Run the program!
Finally we simply run our program by instantiating our class and running the algorithm. Make sure to pass in the number of lineups you want to create (I have set it at 150, but you can put in any number you want from 1 to 150), and the duration in seconds if you wanted to change it from the default 60 seconds runtime. 


```
if __name__ == "__main__":
	#specify the number of lineups to generate (from 1 to 150), and how long to let the program run for (optional)
	#the default duration is 60 seconds.
	#I use runtime instead of # of mutations b/c I think it is more intuitive.
	g = GeneticNHL(num_lineups=150)
	g.load_roster()
	g.run()
	g.save_file()
```


To actually run the program type the following command on your terminal/command prompt


```
$ python genetic_algorithm.py
```

>**Note:** Make sure you run the program from the same folder/directory that contains the DKSalaries.csv and python code.

That's it, we just made a genetic algorithm for DraftKings NHL contests! It is easy to modify the code to work with other sports, but hockey is my favorite, so that's what we have. 

As I caution above, I would be very hesitant to use this exactly as-is to enter into contests. To be successful at daily fantasy sports requires a targeted approach for many different variables. For example, by design, this algorithm doesn't include any additional constraints like line stacking or variance controls. That makes this algorithm unsuitable for many tournament styles, but if you absolutely must use it I would stick to small cash games like 50/50's and Double Ups, as that is where it could provide some value. 

In future posts, I will show you how you can make better, more accurate player projections, and I will walk through how to build a more robust optimizer in python - something we can use to add in many more constraints to build more complex lineups suitable for larger tournament styles. Hopefully soon you will start to understand some of the methods the pros use to win, but for now you can try running the GA and see some of the results. All of the code above can been downloaded with additional comments at my Github [here](https://github.com/jnederlo/daily_fantasy_genetic_algorithm). Enjoy!

Jarvis Nederlof
LineCruncher | FantasyHacker

