This document describes the process of collecting, processing and plotting ADS-B data
received from nearby aircraft. Contact reddit.com/u/aarkebauer with any questions.

------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------

Overview

I use a Raspberry Pi to collect and store the raw ADS-B data in a file. Data is collected
using an inexpensive USB SDR antenna and is stored in the data file at 5 second intervals.
A wide array of such antennas are available, and the one I use is the NooElec NESDR Mini
SDR & DVB-T USB Stick.
(http://www.nooelec.com/store/computer-peripherals/usb-ota-receivers/dvb-t-receivers/nesdr
- mini-rtl2832-r820t.html)

I then transfer this data file to my computer, where I have a bash script that deletes the
'data' folder in the current directory (explained below) and then runs a python program
(dump1090plot.py) that takes creates a new file containing all data collected for each
individual aircraft. The names of these files are the ICAO hex codes of the aircraft, and
they are stored in a folder labeled 'data'. The program then looks at each of these files,
makes lists of all position values (latitude, longitude, altitude) and filters out any
"bad points," which I define as any points at which either altitude, latitude, or
longitude data was NOT collected. The same python program then plots each of these lists
of positions using matplotlib.

There are several options that can be set in this program. They are described below, and
are contained (with descriptions) within the program itself around line 155.

Detailed descriptions of the setup and processes that occur are presented below.

------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------

Raspberry Pi

Several items must be installed on the Raspberry Pi prior to collecting data. First,
I installed dump1090, which will decode the ADS-B data obtained by the receiver (so named
because ADS-B is broadcast at 1090 MHz). The following is the process I used to install
this via command line on the Pi:

sudo apt-get update
sudo apt-get upgrade
sudo su				# to avoid writing sudo at the beginning of the following 6 commands
<The following is one line>
printf 'blacklist dvb_usb_rtl28xxu\nblacklist rtl2832\nblacklist rtl2830' > /etc/modprobe.d/nortl.conf
apt-get install git-core
apt-get install git
apt-get install cmake
apt-get install libusb-1.0-0-dev  
apt-get install build-essential 
git clone git://git.osmocom.org/rtl-sdr.git
cd rtl-sdr
mkdir build
cd build
cmake ../ -DINSTALL_UDEV_RULES=ON
make
make install										# must be run as root
ldconfig											# must be run as root
cd ~
cp ./rtl-sdr/rtl-sdr.rules /etc/udev/rules.d/		# must be run as root
reboot												# must be run as root
<It will reboot>
rtl_test -t
cd ~ 
git clone git://github.com/MalcolmRobb/dump1090.git
cd dump1090
make
apt-get install pkg-config							# must be run as root
make


After this is installed, I would suggest reading through the README.md file in the
dump1090 directory. This contains many useful commands and additional information.

To run dump1090 in the terminal window and collect ADS-B in real time from nearby
aircraft, run the following command while in the dump1090 directory:

./dump1090 --interactive

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

Next, I installed an Apache web server on the Pi. Although an internet connection is NOT
required for any part of this process, the data obtained from dump1090 is stored at
http://localhost:8080/data.json and to access it, you need to be able to navigate to this
page.

The following is the process I used to install this via command line on the Pi:

sudo apt-get install apache2 php5 libapache2-mod-php5

sudo service apache2 restart

Now you can view dump1090 as follows:
	On RasPi: navigate to localhost:8080
	On computer (if connected to internet): navigate to [pi's ip address]:8080
	To view the latest raw data: add /data.json to the end of the above addresses
	Make sure when starting dump1090 on the Pi to use the --net command
	i.e. start with this command: ./dump1090 --interactive --net
	
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

With these two items now installed, I wrote the following small program in python and
saved it to the home directory (/home/pi) as adsb.py:


import urllib
import re # for regular expressions

uf = urllib.urlopen('http://localhost:8080/data.json') # this is where the dump1090 data is stored
w = open("data.dat", 'a');

for line in uf.read(): # read source code
	w.write(line);

w.close()


This program is used to collect (only) the latest set of ADS-B data from the file
described above. It then writes this to a file called "data.dat" in the same directory.

I also wrote the following script in order to delete the previous data.dat file (This is
VERY important, as the above program simply appends new data to the old file, so if it
is not deleted old data will be plotted in addition to the new) as well as run the above
program every 5 seconds:


#! /bin/sh
cd dump1090 &&
./dump1090 --net &
sleep 2 &&
cd &&
rm -rf data.dat &&
while true
do
	sudo python adsb.py &&
	sleep 5
done

exit


I named this myscript.sh in the home directory (/home/pi) and made it executable
(using this command): chmod +x myscript.sh

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

I then start the script (begin recording data) using the following command: ./myscript
(Make sure the antenna is plugged into a USB port!)

To kill stop collecting data, I simply kill the script with control+C (elegant, I know).

Note: The file 'data.dat' grows at ~1.5 MB per hour for light to moderate air traffic.

------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------

Computer (OS X, but it should work for anything)

Make sure to have python and matplotlib installed on your computer. Instructions for
installation of these can be found fairly easily on the internet, so they are not included
here.

First, I transferred the file 'data.dat' from the Raspberry Pi to my computer. There are
a few methods of doing this, but I ssh'd into my Pi from my computer (instructions for
doing so are also readily available by doing some googling.) and used FileZilla for the
transfer.

In the SAME directory as the file titled 'data.dat' (it is important that this file name
is maintained), I wrote a python program titled 'dump1090plot.py' and a bash script titled
'plot.sh'. (Again, important that these file names are maintained if you use the same
procedure as the one I describe here)

The program (~ 400 lines) and script (6 lines) are copied and pasted in this document, and
if this is on github, hopefully are able to be downloaded easily.



TO RUN THE PROGRAM AND MAKE THE PLOT: simply execute plot.sh from the command line (in
the same directory as the script, run this command: ./plot.sh) Remember first to make the
script executable by running this command: chmod +x plot.sh. Instructions for making
an OS X application are included below.

If any crazy flight paths appear, dump1090 may have decoded information incorrectly.
To remedy this, delete all lines in 'data.dat' that contain the hex code of that flight.
The hex code can often be determined by looking at the flight number on the plot, then
going to 'data.dat' and looking at the hex number that corresponds to this flight number.
Running the following in the command line will delete all lines containing the hex number:

grep -v [hex number] data.dat > data.dat



Note: If you don't use this script, make sure a folder titled 'data' is present in the
same directory.

There are several settings within the program, dump1090plot.py, that can be altered
to customize the plot and its rotation.


The first set of settings (~ line 153) contain the following:

*****IMPORTANT*****IMPORTANT*****IMPORTANT*****IMPORTANT*****IMPORTANT*****IMPORTANT******
The first settings are current latitude and longitude. These are used to center the plot
on your location. It is important that these are correct in order to obtain a reasonable
plot.

The rest of the settings in this set are described in the program, but are repeated here
for convenience:

	# SET THESE LINES TO CURRENT LATITUDE AND LONGITUDE TO CENTER GRAPH
	currentlatitude = 40.75306		# Home is 40.75306
	currentlongitude = -96.74417	# Home is -96.74417

	# If the following is set to True it will plot from z = 0
	# If set to False it will plot from z = [minimum altitude] - 1000 ft
	plot_from_ground_level = False

	# The following is used to only plot flights under a certain altitude
	# Set this to float('inf') in order to see all flights
	only_plot_flights_under_this_altitude = float('inf') # to plot all: float('inf')

	# Setting this to True prints out the flight numbers for each plotted line
	# Setting this to False does not display flight numbers
	display_flight_number_labels = True

	# Set the following to True to plot planes WITHOUT flight number data
	# Set it to False to plot ONLY those planes WITH flight number data
	plot_planes_without_flight_numbers = False

	# If set to True planes with flight numbers containing only numbers
	# (e.g. 1653) will be displayed. If set to False then ONLY flight
	# numbers beginning with letters (e.g. UAL1653) will be displayed
	plot_planes_with_number_only_flight_numbers = True

	# Values used to change plot animation parameters are defined below #

	# If set to True all flight numbers will be printed to a document
	# titled flight_numbers.dat. If false, it will not make this file
	print_flight_number_file = False


The next set of settings (~ line 378) deal with the animation of the plot (it rotates):

	# ANIMATION PARAMETERS:
	speed = 1.5 # This is the speed at which the plot rotates. *** Higher numbers = slower ***
	revolutions = 3 # This is the number of revolutions that the plot will complete

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

The python program 'dump1090plot.py' does several things. First (in the createfiles
function), it takes the 'data.dat' file and makes a file for each aircraft that was picked
up. The file is stored in the folder titled 'data' and the filename is the ICAO hex code
of the aircraft. In the file is stored the following information (on separate lines)

Hex code
Flight number
Latitude
Longitude
Altitude
Track (Heading)
Speed

for each time (remember, collected data was written to 'data.dat' every five seconds)
the dump1090 program collected a valid latitude, longitude, AND altitude. An empty
line separates each set of data.

The program then (in the the plot function) opens each of these files (one by one) and
puts the contained data into several dictionaries, called allalts, alllats, allflights, 
and alllons. The keys are just ascending integers (one for each plane), and the values
are lists of the altitudes, latitudes, flight numbers, and longitudes, respectively, for
each plane. Null elements (flights that produced NO valid latitudes, longitudes AND
altitudes) are removed from these dictionaries.

Each of the lists within these dictionaries are used as the x, y, and z coordinates to
plot lines representing the flight paths of each airplane. Latitudes are x values,
longitudes are y values, and altitudes are z values.

Each flight is thusly plotted. Circles representing 60 and 120 miles (roughly, as I know
there is no standard degree:latitude or longitude conversion) as well as a z axis (with
altitude value labels) and cardinal directions are drawn on the plot as well.

The plot is scaled via a makeshift method described in detail in the comments of the
program, around line 320.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

The bash script 'plot.sh' does a few simple things. It deletes the 'data' folder and its
contents so that the old flights are not plotted again, deletes the old flight_numbers.dat
file (if this setting is enabled in the program, as described above), creates a new empty
folder titled 'data' and then runs dump1090plot.py

If you don't use this script, these steps are still important to do before running
dump1090plot.py.

The script is copied here for convenience:


#!/bin/bash

rm -rf data &&
rm -rf flight_numbers.dat &&
mkdir data &&
python dump1090plot.py

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

Turning 'plot.sh' into a OS X application.

I created an application that executes a script, called 'plot2.sh'. This is identical
in function to 'plot.sh' but includes an additional step necessary for function as an
application. It is copied here for convenience, but make sure to edit the 7th line to
contain the path to the folder that data.dat, dump1090plot.py, etc. are stored in:


#!/bin/bash

# This script is run by the Dump1090Plotter app
# It includes an additional cd to the dump1090plot folder

cd &&
cd [FOLDER CONTAINING data.dat, dump1090plot.py, ETC.] &&
rm -rf data &&
rm -rf flight_numbers.dat &&
mkdir data &&
python dump1090plot.py


Thomas Aylott came up with a clever “appify” script that allows you to easily create Mac
apps from shell scripts. It is copied here for convenience:


#!/usr/bin/env bash

APPNAME=${2:-$(basename "$1" ".sh")}
DIR="$APPNAME.app/Contents/MacOS"

if [ -a "$APPNAME.app" ]; then
  echo "$PWD/$APPNAME.app already exists :("
  exit 1
fi

mkdir -p "$DIR"
cp "$1" "$DIR/$APPNAME"
chmod +x "$DIR/$APPNAME"

echo "$PWD/$APPNAME.app"


First, save the script to the /usr/local/bin directory and name it appify (no extension).

Then enter the following command in command line: sudo chmod +x /usr/local/bin/appify
to make appify executable without root privileges.

Then, the following command can be run to create the app (from the directory containing
'plot2.sh' (and all other necessary files, described above)):

appify plot2.sh "Your App Name"


You can create a custom icon by right clicking on the app and selecting Get Info. Copy an
image to your clipboard, click on the icon at the top left of the Get Info box, and paste
the image.


------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------


The following is the code for dump1090plot.py:




import re, os
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

def main():
	createfiles() 	# takes the full data.dat and breaks it down into a single file for each
					# plane (based off of icao hex code)
	plot()			# pull data from each file and make a 3d plot of each plane's path based
					# on the latitude, longitude, and altitude data they contain

def createfiles():
	dat = open("data.dat", 'rU');
	
	flag = { }  # create dictionary to be used to store hex codes. If a hex code has already been stored,
				# then the data file must exist for that code, so don't write it, just append to it
	i = 0;		# keys for dictionary (arbitrary, serve no purpose)
	
	for line in dat: # read source code
	
		hexdat = re.search('("hex":")([\w|\d]*)', line);
		if hexdat:
			hexdata = hexdat.group(2);
			flightdat = re.search('("flight":")([\w|\d]*)', line);
			if flightdat:
				flightdata = flightdat.group(2)
			latdat = re.search('("lat":)([-]*[\d]*\.[\d]*)', line);
			latdata = latdat.group(2)
			londat = re.search('("lon":)([-]*[\d]*\.[\d]*)', line);
			londata = londat.group(2)
			altdat = re.search('("altitude":)(\d*)', line);
			altdata = altdat.group(2)
			trackdat = re.search('("track":)(\d*)', line);
			trackdata = trackdat.group(2)
			speeddat = re.search('("speed":)(\d*)', line);
			speeddata = speeddat.group(2)
	
			filename = "data/" + str(hexdata)# + ".dat"
	
			if str(hexdata) in flag.values(): 	# If a hex code has already been stored, then the data file
												# must exist for that code, so don't write it, just append
				wrfile = open(filename, 'a');
				wrfile.write("\n")
			else:
				wrfile = open(filename, 'w');
	
			wrfile.write(hexdata + "\n");
			if flightdat:
				wrfile.write(flightdata + "\n");

			# if latitude, longitude, or altitude data is a 0 (i.e. not present), don't write
			# any of the three - this cleans up the lists to eliminate bad data points when plotting
			if float(latdata) != 0:
				if float(londata) != 0:
					if float(altdata) != 0:
						wrfile.write(latdata + "\n");
						wrfile.write(londata + "\n");
						wrfile.write(altdata + "\n");
			else:
				wrfile.write("\n");
				wrfile.write("\n");
				wrfile.write("\n");

			wrfile.write(trackdata + "\n");
			wrfile.write(speeddata + "\n");
			wrfile.close()
	
			flag[i] = str(hexdata) # update dictionary
			i +=1

def plot():

	############################################################################
	# This is where data is extracted from files and put into lists for plotting
	############################################################################

	allalts = { }
	alllats = { }
	alllons = { }
	allflights = { }
	counter = 0

	for file in os.listdir("data/"): # If an empty file named .DStore or something exists, it may not work - ?
		filename = "data/" + file;
		workingfile = open(filename, 'rU')


		worker = open(filename, 'rU'); 	# must open the file under a different name to count number of lines
		length = 0;
		for line in worker: 	# count number of lines in file to determine how long lists (below) should be
			length +=1;


		flightnumberlines = [x*8+1 for x in range(0, length)]
		latitudelines = [x*8+2 for x in range(0, length)]
		longitudelines = [x*8+3 for x in range(0, length)]
		altitudelines = [x*8+4 for x in range(0, length)]

		# initialize these as the empty list
		latlist = list()
		longlist = list()
		altlist = list()
		flightnumberlist = list()

		i = 0
		for line in workingfile:
			if i in latitudelines:
				latlist.append(line[0:len(line)-1])
			if i in longitudelines:
				longlist.append(line[0:len(line)-1])
			if i in altitudelines:
				altlist.append(line[0:len(line)-1])
			if i in flightnumberlines:
				flightnumberlist.append(line[0:len(line)-1])
			i += 1

		#print altlist
		#print latlist
		#print longlist
		#print flightnumberlist

		# remove all null elements from the lists (planes that produced 0 complete position data points)
		altlist = map(float, filter(None, altlist))
		latlist = map(float, filter(None, latlist))
		longlist = map(float, filter(None, longlist))
		flightnumberlist = filter(None, flightnumberlist)

		allalts[counter] = altlist
		alllats[counter] = latlist
		alllons[counter] = longlist
		allflights[counter] = flightnumberlist

		counter += 1

		workingfile.close()
		worker.close()

	#print allalts
	#print alllats
	#print alllons
	#print allflights



	##########################################################################################################
	##########################################################################################################
	#										THIS IS WHERE PLOTTING OCCURS								     #
	##########################################################################################################
	##########################################################################################################



	#####################################################################
	#				THESE VALUES ARE USED TO FORMAT THE PLOTS			#
	#####################################################################

	# SET THESE LINES TO CURRENT LATITUDE AND LONGITUDE TO CENTER GRAPH
	currentlatitude = 0.0000
	currentlongitude = -0.0000
	# If the following is set to True it will plot from z = 0
	# If set to False it will plot from z = [minimum altitude] - 1000 ft
	plot_from_ground_level = False

	# The following is used to only plot flights under a certain altitude
	# Set this to float('inf') in order to see all flights
	only_plot_flights_under_this_altitude = float('inf') # to plot all: float('inf')

	# Setting this to True prints out the flight numbers for each plotted line
	# Setting this to False does not display flight numbers
	display_flight_number_labels = True

	# Set the following to True to plot planes WITHOUT flight number data
	# Set it to False to plot ONLY those planes WITH flight number data
	plot_planes_without_flight_numbers = False

	# If set to True planes with flight numbers containing only numbers
	# (e.g. 1653) will be displayed. If set to False then ONLY flight
	# numbers beginning with letters (e.g. UAL1653) will be displayed
	plot_planes_with_number_only_flight_numbers = True

	# Values used to change plot animation parameters are defined below #

	# If set to True all flight numbers will be printed to a document
	# titled flight_numbers.dat. If false, it will not make this file
	print_flight_number_file = False
	
	#####################################################################
	#####################################################################
	#####################################################################



	plt.ion() # this is necessary for animating (spinning in this case) the plot - it must stay here!!!
	
	fig = plt.figure(figsize=(15,9)) # this figure size fits well to macbook pro 13.3" screen
	ax = fig.add_subplot(111, projection='3d')



	minimumaltitude = filter(None, allalts.values())
	minimums = list() # initialize list of minimum altitudes of each plane
	for item in minimumaltitude:
		minimums.append(min(item))
	minimumaltitude = min(minimums) # find the absolute lowest altitude to be used as the bottom of the graph
	
	if plot_from_ground_level != True: # Described above, used to either plot from 0 up or [minalt - 1000] up
		floor = minimumaltitude - 1000
	else:
		floor = 0



	# these 2 circles are used to give an idea of the locations of the planes - each is ~120 miles in diameter
	xcircle = range(-100,101)
	xcircle = [float(item) / 100 for item in xcircle] # create a list from -2 to 2 with increment .1
	ycirclepos = [pow(pow(1,2) - (pow(item, 2)), .5) for item in xcircle]
	ycircleneg = [-pow(pow(1,2) - (pow(item, 2)), .5) for item in xcircle]
	ax.plot(xcircle, ycirclepos, floor, color='0.7')
	ax.plot(xcircle, ycircleneg, floor, color='0.7')
	# place the 120mi label at a 45 degree angle (in the NE quadrant of the plot)
	ax.text(-((pow(2,0.5)/2)-.05),(pow(2,0.5)/2)-.05,floor,"60 mi", fontsize=8, color='0.7')

	xcircle2 = range(-200,201)
	xcircle2 = [float(item) / 100 for item in xcircle2] # create a list from -2 to 2 with increment .1
	ycirclepos2 = [pow(pow(2,2) - (pow(item, 2)), .5) for item in xcircle2]
	ycircleneg2 = [-pow(pow(2,2) - (pow(item, 2)), .5) for item in xcircle2]
	ax.plot(xcircle2, ycirclepos2, floor, color='0.7')
	ax.plot(xcircle2, ycircleneg2, floor, color='0.7')
	# place the 120mi label at a 45 degree angle (in the NE quadrant of the plot)
	ax.text(-(pow(2,0.5)-.05),pow(2,0.5)-.05,floor,"120 mi", fontsize=8, color='0.7')


	all_plotted_lats = list()
	all_plotted_lons = list()
	all_plotted_alts = list()
	#feetperdegree = 362776; # to measure altitude in (roughly) degrees latitude
	for plot in range(len(allalts.values())):
		xx = alllats.values()[plot]
		xx = [item - currentlatitude for item in xx] # center the graph around current latitude
		xx = [-item for item in xx] # flip everything over the x axis so directions line up
									# Think about this like you're looking up at the planes
									# from below - positions of the cardinal directions are swapped
									# compared to looking down from above as you are in the plot
		yy = alllons.values()[plot]
		yy = [item - currentlongitude for item in yy] # center graph around current longitude
		zz = allalts.values()[plot]
		#zz = [item / feetperdegree for item in zz] # to measure altitude in (roughly) degrees latitude
		
		# obtain the flight number, if any
		flightnum = allflights.values()[plot]
		if len(flightnum) != 0:
			# If the flight number is only a number (e.g. 1653), then plot/don't plot according to setting
			if plot_planes_with_number_only_flight_numbers != True:
				chars = []
				for c in flightnum[0]:
					chars.append(c)
				letters = ["A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P",
				 "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"]
				if chars[0] in letters:
					flightnum = flightnum
					if print_flight_number_file == True:
						flight_num_file = open("flight_numbers.dat", 'a')
						flight_num_file.write(flightnum[0] + "\n")
				else:
					flightnum = " "
			else:
				flightnum = flightnum
				if print_flight_number_file == True:
						flight_num_file = open("flight_numbers.dat", 'a')
						flight_num_file.write(flightnum[0] + "\n")
		else:
			flightnum = " "



		###############
		# plot the data
		###############

		# x-axis : latitude (degrees from current location, as set below)
		# y-axis : longitude (degrees from current location, as set below)
		# z-axis : altitude (ft)
		
		if len(xx) != 0: 				# don't attempt to plot empty lists
		
			# if the flight's minimum altitude is under the set threshold, plot the flight
			if only_plot_flights_under_this_altitude >= min(allalts.values()[plot]):
				if plot_planes_without_flight_numbers != True:	# this is set/described above
					if flightnum != " ":						# plot only planes WITH flight numbers
						ax.plot(xx, yy, zz)			# use ax.scatter(xx, yy, zz) for a scatter plot, etc.
						
						# The following statements keep track of all lats, lons, and alts that are PLOTTED
						# This is used for scaling the final plot, as described below
						for item in xx:
							all_plotted_lats.append(item)
						for item in yy:
							all_plotted_lons.append(item)
						for item in zz:
							all_plotted_alts.append(item)
						
						if display_flight_number_labels == True:
							ax.text(xx[0],yy[0],zz[0], " "+flightnum[0], fontsize=6)
				else:
					ax.plot(xx, yy, zz)			# use ax.scatter(xx, yy, zz) for a scatter plot, etc.
					
					# The following statements keep track of all lats, lons, and alts that are PLOTTED
					# This is used for scaling the final plot, as described below
					for item in xx:
						all_plotted_lats.append(item)
					for item in yy:
						all_plotted_lons.append(item)
					for item in zz:
						all_plotted_alts.append(item)
					
					if display_flight_number_labels == True:
						ax.text(xx[0],yy[0],zz[0], " "+flightnum[0], fontsize=6)


	# *** all_plotted_lats, all_plotted_alts, all_plotted_lons contain only values of PLOTTED planes *** #
	# The following obtains min and max latitude and longitude to be used in setting frame limits
	# of the plot - a makeshift way of scaling
	minimumlat =  min(all_plotted_lats)
	maximumlat =  max(all_plotted_lats)
	minimumlon =  min(all_plotted_lons)
	maximumlon =  max(all_plotted_lons)
	# The z-axis autoscale seems to be good enough, but these are here anyway for possible future use
	minimumalt =  min(all_plotted_alts)
	maximumalt =  max(all_plotted_alts)
	
	#ax.set_zlim(bottom=floor)

	# to keep the graph centered, the x and y limits must all be the same. So, I chose the largest
	# relative lat OR lon to be the absolute limit so that every point will stay within the frame
	# note: the z autoscale seems to be good enough
	s = max(abs(minimumlat), abs(minimumlon), abs(maximumlat), abs(maximumlon))
	ax.set_xlim(left=-s, right=s)		# x is latitude
	ax.set_ylim(bottom=-s, top=s)		# y is longitude

	#plt.show() # this is used for a static graph





	# make lines for x and y axes that extend slightly past the 120mi (2 degrees) marking circle
	xlimits = ax.get_xlim()
	ylimits = ax.get_ylim()
	ax.plot([float(xlimits[0])-.5, float(xlimits[1]+.5)], [0,0], [floor, floor], color='0.7')
	ax.plot([0,0], [float(ylimits[0])-.5, float(ylimits[1]+.5)], [floor, floor], color='0.7')

	# label the cardinal directions
	ax.text(0, s+.55, floor, " E", color='0.7')
	ax.text(0, -s-.55, floor, " W", color='0.7')
	ax.text(s+.55, 0, floor, " S", color='0.7')
	ax.text(-s-.55, 0, floor, " N", color='0.7')

	zticks = ax.get_zticks() # get the values that are displayed on the z axis by default
	ax.plot([0,0], [0,0], [floor,max(zticks)], color='0.7') # line for z axis


	# this prints a the default z-axis labels next to the new, centered, z axis
	itemnumber = 0
	for item in zticks:
		if (itemnumber % 2 != 0): # display every other label from the default z-axis labels
			ax.text(0,0,item," " + str(int(item)), fontsize=8, color='0.7') # z axis drawn at x=0, y=0
		itemnumber+=1

	plt.axis('off')			# turn ALL axes off



	# this is to rotate the plot at an elevation of 30 degrees through a full 360 degrees azimuthally
	# it quits after rotating 360 degrees (or whatever the upper limit of the range is)

	########################################################################################
	########################################################################################
	# ANIMATION PARAMETERS:
	speed = 1.5 # This is the speed at which the plot rotates. *** Higher numbers = slower ***
	revolutions = 3 # This is the number of revolutions that the plot will complete
	########################################################################################
	########################################################################################

	azimuths = range(0,int(((360*revolutions)*speed)+1))
	azimuths = [float(item) / speed for item in azimuths]
	for azimuth in azimuths:
		ax.view_init(elev=45, azim=azimuth)
		plt.draw()






# This is the standard boilerplate that calls the main() function initially
if __name__ == '__main__':
	main()
