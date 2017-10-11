---
layout: post
title: Did my flight just get cheaper?
subtitle: Monitoring Southwest Airlines flight prices with RSelenium and Rvest
---

I am a fairly frequent flyer, and my preferred airline is [Southwest](http://www.southwest.com). I love that their trips are booked in one-way segments, their open seating system, that I can check two bags for free, and most importantly, that I can change or cancel my flights at any time without penalty.

Changing flights without penalty means that I can book a trip, and if that trip becomes cheaper at any time, I can cancel my previous flight and rebook the same flight, and pocket the difference. However, Southwest does not provide an alert that lets you know when a price has dropped – you have to check this manually.


## Automating the price check
In order to make sure I get the cheapest prices for all my Southwest flights, I created an R script to automatically check the flight prices for flights I have purchased. If a lower price is available, I receive an email with the flight info and my potential savings.

There are four major steps that we will go through below in more detail:
<ol><li value="1">  Open a web browser, and drive the browser using R via RSelenium</li>
  <li>  Scrape the flight schedule using rvest </li>
  <li>  Parse the resulting table using dplyr and stringr to check for price changes </li>
  <li>  Send an email alert using mailR if the price has decreased </li>
</ol>


### Defining our initial purchase
For this post we will use a simple example and track just one flight. Let’s say I purchased a flight from Raleigh, NC to Portland, OR on December 9th for $204, departing at 7:15am and arriving at 12:00pm. First we'll load the necessary packages and define our purchased flight details within our environment. 

<pre><code class="language-r line-numbers">#load packages
library(RSelenium)
library(rvest)
library(tidyverse)
library(stringr)
library(mailR)
library(taskscheduleR)

#define current flight info
purchased_departure_airport_code <- "RDU"
purchased_arrival_airport_code <- "PDX"
purchased_departure_date <- "12/09"
purchased_departure_time <- "7:15"
purchased_arrival_time <- "12:00"
purchased_price <- 204
</code></pre>


## Driving a web browser programmatically
Now that we have our details of our previous purchase defined, let’s use RSelenium to input those details in Southwest’s website. First, we have to define our remote driver, which will allow us to control Chrome using R and browse the web via R commands. Then we can navigate to the Southwest homepage.

<pre><code class="language-r line-numbers">#define remote driver and open browser
rD <- rsDriver(verbose = FALSE)
remDr <- rD$client

#navigate to southwest website homepage
remDr$navigate("https://www.southwest.com")
Sys.sleep(5) #wait for the page to load
</code></pre>

![alt text](/img/southwest/southwest_homepage.JPG "Southwest Homepage")


### Entering inputs onto a webpage
Let’s now interact with the webpage, entering the flight info for the flight that was already purchased. My favorite way to figure out where on the website to pass each input is by using [selector gadget](http://www.selectorgadget.com) to investigate the website’s CSS.

<pre><code class="language-r line-numbers">#click one-way radiobutton
oneway_radiobutton <- remDr$findElement(using = 'css selector', "#trip-type-one-way")
oneway_radiobutton$clickElement()
  
#enter departure airport code
departure_airport <- remDr$findElement(using = 'css selector', "#air-city-departure")
departure_airport$clearElement()
departure_airport$sendKeysToElement(list(purchased_departure_airport_code))

#enter arrival airport code
arrival_airport <- remDr$findElement(using = 'css selector', "#air-city-arrival")
arrival_airport$clearElement()
arrival_airport$sendKeysToElement(list(purchased_arrival_airport_code))

#enter departure date
departure <- remDr$findElement(using = 'css selector', "#air-date-departure")
departure$clearElement()
departure$sendKeysToElement(list(purchased_departure_date))

#click the search button
search <- remDr$findElement(using = 'css selector', "#jb-booking-form-submit-button")
search$clickElement()
Sys.sleep(5)
</code></pre>


Before we run our ‘search’ command the booking window on the frontpage looks like this:
![alt text](/img/southwest/flight_input.JPG "Flight Booking Input")


## Scraping Flight Schedules
Now that we have navigated to the day and airports that match our purchased ticket, we can use the page's HTML to create a data frame of the flight schedule in R. This is the flight schedule for that day:

![alt text](/img/southwest/flight_schedule2.JPG "Flight Schedule")

### Parsing the HTML
Again, we will use selector gadget to identify the elements in the CSS that are important to us. We can use the rvest package to parse the HTML, and dplyr and stringr to transform the HTML text into the variables that we want.

<pre><code class="language-r line-numbers">#read page html
southwest_html <- read_html(remDr$getPageSource()[[1]])

#scrape and clean departure times
flight_departure_times <- southwest_html %>% 
  html_nodes(css = ".depart_column") %>% 
  html_text(trim=TRUE) %>% 
  str_replace_all("[\r\n\t]" , "") %>% 
  str_sub(start=1, end=8) %>% 
  str_replace_all("[AMPM]" , "") %>% 
  str_trim()

#add departure times to flight schedule data frame
flight_schedule <- tibble(flight_departure_times)

#add arrival times to flight schedule data frame
flight_schedule$flight_arrival_times <- southwest_html %>% 
  html_nodes(css = ".arrive_column") %>% 
  html_text(trim=TRUE) %>% 
  str_replace_all("[\r\n ]" , "") %>% 
  str_replace_all("[AMPM]" , "") %>% 
  str_trim()

#add business select price to flight schedule data frame
flight_schedule$business_price <- southwest_html %>% 
  html_nodes(css = ".price_column:nth-child(6)") %>% 
  html_text(trim=TRUE) %>% 
  str_replace_all("[\r\n\t]" , "") %>% 
  str_extract("[\\$][0-9]+") %>% 
  str_replace_all("[\\$]" , "") %>% 
  str_trim() %>% 
  as.numeric

#add anytime price to flight schedule data frame
flight_schedule$anytime_price <- southwest_html %>% 
  html_nodes(css = ".price_column:nth-child(7)") %>% 
  html_text(trim=TRUE) %>% 
  str_replace_all("[\r\n\t]" , "") %>% 
  str_extract("[\\$][0-9]+") %>% 
  str_replace_all("[\\$]" , "") %>% 
  str_trim() %>% 
  as.numeric

#add wanna get away price to flight schedule data frame
flight_schedule$getaway_price <- southwest_html %>% 
  html_nodes(css = ".price_column:nth-child(8)") %>% 
  html_text(trim=TRUE) %>% 
  str_replace_all("[\r\n\t]" , "") %>% 
  str_extract("[\\$][0-9]+") %>% 
  str_replace_all("[\\$]" , "") %>% 
  str_trim() %>% 
  as.numeric
</code></pre>

After pulling the HTML and performing some transforming the resulting data, our final cleaned data frame looks like this:
![alt text](/img/southwest/flight_schedule_dataframe.JPG "Flight Schedule Dataframe")


## Checking the flight price
Now that we have our flight schedule in R, we will want to match the flight that we have already purchased and check its price. We will do this by filtering the data frame to only the observation that matches our purchased flight departure and arrival times. 
Finally, we can take the minimum price for that flight, and store it a separate variable called _current_lowest_price_.

<pre><code class="language-r line-numbers">#filter flight schedule to look at only previously purchased flight
my_flight_schedule <- flight_schedule %>% 
  filter(flight_departure_times == purchased_departure_time) %>% 
  filter(flight_arrival_times == purchased_arrival_time) %>% 
  mutate(low_price = (min(business_price, anytime_price, getaway_price))) 

#create variable containing lowest price for previously purchased flight
current_lowest_price <- my_flight_schedule$low_price
</code></pre>


## Creating an email alert
Next, we’ll set up an email alert which will send us an email in the event that the prices drops. To send the email alert, we will use the [mailR]() package. We will wrap the sendMail command in an ifelse statement, so that we only receive an email if the current lowest price is lower than the purchased ticket price. 
The subject and body of the email can say whatever we want, but I’ve kept it simple for now. Additionally, the _sender_, _recipient_, and _pw_ variables were defined locally. 

<pre><code class="language-r line-numbers">#send email alert if lower price is available
ifelse(purchased_price > current_lowest_price, 
  send.mail(from = sender,
          to = recipient,
          subject = paste0(("SOUTHWEST LOWER FARE ALERT: "), 
                           purchased_departure_airport_code, (" to "), 
                           purchased_arrival_airport_code, (" on "), 
                           purchased_departure_date, (" has decreased by $"),
                           purchased_price-current_lowest_price),
          body = "Rebook flight at www.southwest.com.",
          smtp = list(host.name = "smtp.gmail.com", port = 465, 
                      user.name = sender,            
                      passwd = pw, ssl = TRUE),
          authenticate = TRUE,
          send = TRUE), 
  paste("no lower price available"))
</code></pre>

The current lowest price for our flight is $135, which is lower than our purchased flight price of $204. Let’s check our email!

![alt text](/img/southwest/email_alert.JPG "Email alert successful!!")

## Scheduling the Script
As a final step, we want our script to check for prices daily. To do this, we can leverage the [taskscheduleR]() package, and the associated addon to RStudio. Using the GUI we decide which script to use and how often we want it to run. 

![alt text](/img/southwest/taskscheduleR.JPG "Schedule script to run daily")

We now have a program that can automatically check for cheaper Southwest flight prices without us having to lift a finger. Happy flying everyone!
