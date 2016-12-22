---
layout: post
title: Predicting Genre using Song Lyrics (part 1)
---

I’ve always loved music. In order to learn more about text mining, I thought it would be interesting to see if you could predict a song’s genre from its lyrics.

If we’re going to try to predict a song’s genre using its lyrics, one big issue to address first is the subjective nature of our target variable, _genre_. For example, is Taylor Swift a pop artist, or a country singer? Similarly, are alternative and heavy metal separate genres, or can you just classify both as just ‘Rock’?  Classifying a song within a genre is a subjective exercise.
 
##Gathering the data
Luckily, I was able to avoid listening to hundreds of songs and making my own subjective assessments on genre, thanks to a website called [Songlyrics](http://www.songlyrics.com/news/top-genres/country-music). Songlyrics.com provided lyrics for the top 100 songs in 6 different genres: Christian, Country, Rap, Pop, R&B, and Rock. While there was no information from Songlyrics on how they made their decisions (How did they determine genre? Top 100 songs by what metric?) the songs passed the sniff test: Hank Williams was in the Country genre, and Guns N’Roses were in the Rock genre.
Now that we have a data source, we need to actually capture that data. There were 100 songs in 6 different genres, so our dataset will have 600 rows, where each row is a song. To begin, our dataset will have 3 columns: songinfo (including artist name and song title), lyrics, and genre.
To create our dataset, I wrote a small python program using the [beautiful soup] (https://www.crummy.com/software/BeautifulSoup/bs4/doc/) library to scrape the songinfo, lyrics, and genre for each of the 600 songs. Beautiful Soup is a handy library that will allow you to easily parse HTML tags on websites, allowing you to pull the information you want from a website.
 
First, I imported the relevant libraries and created 3 empty vectors, one for each column of desired data to be stored. I also created a list containing 6 urls, one for each of the genres. There were also three song urls which did not contain content at the time and kept breaking my loop, so they are noted here for omission later.


 ```python
from bs4 import BeautifulSoup
import urllib2
import numpy
import csv
 
lyricsvector = []
genrevector = []
songinfovector = []  #artist and songname
 
# List the URLs here
urllist = [
"http://www.songlyrics.com/news/top-genres/christian/",
"http://www.songlyrics.com/news/top-genres/country-music/",
"http://www.songlyrics.com/news/top-genres/hip-hop-rap/",
"http://www.songlyrics.com/news/top-genres/rhythm-blues/",
"http://www.songlyrics.com/news/top-genres/pop/",
"http://www.songlyrics.com/news/top-genres/rock/"
]

# remove broken links
bad1 = 'http://www.songlyrics.com/charles-wright-the-watts-103rd-street-rhythm-band/miscellaneous/'
bad2 = 'http://www.songlyrics.com/the-beatles/i-want-to-hold-your-hand-lyrics/'
bad3 = 'http://www.songlyrics.com/iggy-pop-and-the-stooges/search-and-destroy-lyrics/'

```

Next, I looped through each genre in urllist and used beautiful soup to parse the html, looking for links to the top 100 songs in each genre. The urls for each song in the top 100 were stored in a new list called songlinks. 


```python
# convert top 100 urls into parseable html
for i in range(0,6):
               url = urllib2.urlopen( urllist[i] )
               doc = url.read()
               soup = BeautifulSoup(doc, 'html.parser')
               div = soup.find( 'div', { 'class': 'box listbox' } )
 
# get genres
               title = soup.title.get_text().encode('ascii', 'ignore').split(' ')
               index100 = title.index('100')
               indexSongs = title.index('Songs')
               genre = ' '.join(title[(index100+1):(indexSongs)]).encode('utf-8')
 
# create list of top 100 songs by genre
               print genre
               songs = div.find_all('a')
               songlinks = []

# create loop to extract song links
               for j in range(0,200): #[0::2]:
                              songlink = songs[j].get('href').encode('ascii', 'ignore')
                              songlinks.append(songlink) #output links to a list called songlinks
 
               songlinks = filter(None, songlinks)
               songlinks = [songlink for songlink in songlinks if (len(songlink.split('/'))==6)]
               if bad1 in songlinks:
                              songlinks.remove(bad1)
               if bad2 in songlinks:
                              songlinks.remove(bad2)
               if bad3 in songlinks:
                              songlinks.remove(bad3)
```


Now that we have the song specific links, it’s time to extract the lyric text from each of those pages, along with some other relevant information. A typical page looks something like [this](http://www.songlyrics.com/the-beatles/yesterday-lyrics/). Using the beautiful soup library again, we were able to capture the songlyrics, genre, and songinfo from each url, and save the results to the empty vectors we created at the outset. After looping through each of the 600 links, I exported the data to a text file for analysis in R. 
 
```python  
# loop through songlinks list to get the actual lyrics
               for k in range(0,len(songlinks)):
                              songurl = urllib2.urlopen( songlinks[k] )
                              songdoc = songurl.read()
                              songsoup = BeautifulSoup(songdoc, 'html.parser')
                              songinfo = songsoup.title.get_text().encode('ascii', 'ignore')
                              print songinfo, 'is number', k
 
                              songdiv = songsoup.find( 'div', { 'id': 'songLyricsDiv-outer' } )
                              lyrics = songdiv.getText().replace("\n", " ").replace("\'", "").replace("\r", " ").encode('utf-8')
 
                              lyricsvector.append(lyrics)
                              genrevector.append(genre)
                              songinfovector.append(songinfo)
 
# export lyrics to tab delimited file
with open('~\TextMining_Lyrics.txt', 'w') as f:
               writer = csv.writer(f, delimiter='\t')
               writer.writerows(zip(songinfovector, lyricsvector, genrevector))
```



