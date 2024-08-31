# 4plebs Archive Scraper

Max Rizzuto | DFRLAB 2022

------
This notebook uses Selenium and Firefox's Gecko Driver to retreave the results of a structured query on the 4chan archive [archive.4plebs.org](archive.4plebs.org).

This notebook has been used for a variety of DFRLab investigations including [QAnon’s hallmark catchphrases evaporating from the mainstream internet](https://medium.com/dfrlab/qanons-hallmark-catchphrases-evaporating-from-the-mainstream-internet-ce90b6dc2c55).

Requires the creation of a "cache" and "results" folder in the notebook's working directory.

------
## Step-by-Step instructions detailing the building process and reasoning:
This was included as part of an internal training session on scraping with Selenium.

This scraper will be built in following order:
* Loading the page / inputting the queries
* Iterating through pages - I like to get this done first.
* Identifying the list of post elements
* Identify the elements within each post
* Iterating through the list of page elements
    * Creating a dictionary output for each page
    * Adding each dictionary to either a list or a central dictionary
* A function that will call all of the above functions, a loop to drive the whole thing.  
* A function that exports the results as a csv

Because there are so many results for our search terms of interest this scraper has got to run totally unattended. This means acknowledging when the all the results from one search have been collected and moving onto the next automatically. 

__Observations:__

_Loading the page_ ---------------------------------------------------------------------------------------------->

Boolean search does not appear to work on 4plebs. This isnt the end of the world, just means its going to take longer to run.
Because we will likely pull the same posts for various search terms, we're going to add the search term to the post results dictionary as metadata for us to de-dup later.

Also, Rather than using the search bar to search for results, we're going to use the url itself (which is sort of structured like an API). 
> ex: [archive.4plebs.org/{board}/search/text/{query}/](_)

Im going to be developing the scraper on a search that only produces a few results, in this case its the search term "reddit rules" which returns 80 results.

Something else that occurs to me is that we need to manage the list of search terms. Feeding each term to the "get_site" function once no more results remain.

Lmao, I just ran the scraper for the first time and it detected the robotic user agent and block the request.
They say that crawling like this is not necessary because everything is backed up on archive.org. This is only partly true and I will continue to work around this.
They caught us because the geckdriver announces itself in the brouser's user-agent. We can generate a fake user agent to appear more authentic.

This worked. I found some code [here](https://stackoverflow.com/questions/29916054/change-user-agent-for-selenium-web-driver), dropped it in and it worked great. 

But now I am on edge. If they put this crawling measure in place there may be others. We need to write the scraper to iteratively save progress so that if we are caught later down the line we dont lose all of our progress. 

_Iterating through pages_ ---------------------------------------------------------------------------------------->

At the bottom of the page I see this very nice, very static "next" button for advancing to the next page. We're going to use that because its not going to move a whole lot.
The list element that the next button is a child to also includes information about its status; When there are no more pages to advance to the li changes from "next" to "next disabled". We're going to use that to tell us when we've reached the last page. 

Selenium's ```find_elements_by_{something}``` is really interesting and useful. When using ```find_element_by_{something}``` (with ```element``` singular) if will return the first matching element. When using ```elements``` plural it will return a list of matches. 

We Figured out sytax for ```get_attribute("class")```. It's frustrating to forget the difference between ```find_element``` and ```get_attribute```. 

I encountered an odd error where the next button could not be clicked itself, but the child element could. You can see in the next button function the variable ```clickable_next_button``` which uses find_element_by_xpath to advance to the child element, at which point I can click.  

_Identifying the list of posts_ --------------------------------------------------------------------------------->

This step has been really easy. The posts are stored in an ```<aside>``` block with the unique class "posts", so we can just find it using ```find_element_by_class_name("posts")```. Note that this is the outer element, we want the inner list, so we do ```find_elements_by_xpath("./*")``` to navagate to the various children elements. 

_Identifying the specific elements from each post that we want to capture_ -------------------------------------->

I stepped through one of the items in the posts lists and pulled out each descrete value of interest. 
I've included the original process here. This was done first to make the second iteration of this process easier:
```
posts[0].find_element_by_tag_name("header").text
posts[0].get_attribute('id')
post_title = posts[0].find_element_by_tag_name('h2').text
post_author = posts[1].find_element_by_class_name("post_author").text
post_tripcode = posts[1].find_element_by_class_name("post_tripcode").text
post_hash = posts[1].find_element_by_class_name("poster_hash").text
post_datetime = posts[1].find_element_by_tag_name("time").get_attribute("datetime")
post_type = posts[1].find_element_by_class_name("post_type").find_element_by_xpath("./*").get_attribute("href")
post_replies = 
post_text = posts[4].find_element_by_class_name("text").text
```

_Iterating through the list of page elements
    * Creating a dictionary output for each page
    * Adding each dictionary to either a list or a central dictionary_ ----------------------------------------->
    
This turned out to be extremely straight forward. You can see the function ```gather_page_results()``` and how it compared to the above code block where I hashed out each of the values we wanted to extract.

I changed the variables into dictionary entries to save memory. you can declare your variables directly into the dictionary, no problem. 

I decided to go for the list of dictionaries because it plays extremely well with pandas, and I already know how it works. You can take the output of that fuction and put it straight into a dataframe, like so: ```pd.DataFrame(gather_page_results())```.

_A function that will call all of the above functions, a loop to drive the whole thing._ ----------------------->

I was preparing for the worst here. After I ran the scraper a couple of times and encountered rate limiting I decided to build in some redundancies to allow us to save progress in the event that we continued to encounter errors.
A consiquence of this is some added complexity in this otherwise pretty simple function. 
You will see the variable ```start_at_page``` is doing some strange things. It is declared in the _Input Parameters_ area below and is designed to allow the crawler to pick up where it left off after a crash inorder to avoide duplicate work. In this function it is called once (and set to be global) at which point it is set to zero, this is so that on the first run it will pick up at _x_ page of archive results, but will start the next search term at zero. In the event that the ```start_at_page``` variable is used users would need to truncate the list of terms to correspond to the values they have not yet captured. 

Down to brass tax---
We've got two vital loops here: 
>An outer loop that iterates though the list of search terms we plan to query. <br>
>It runs the function to gather page results and adds the results to a ```data``` variable and innitiates the inner loop...

>The inner loop is a while loop that continues to run the ```advance_to_next_page``` function and add gathered results to the ```data``` variable. It also creates caches every 5 pages for redundancy.<br>
>The while loop is broken when the next page button is no longer available and the ```advance_to_next_page``` function returns false. At which point the window is abandoned and a new window is opened for the next term.  

_A function that exports the results as a csv_--------------------------------------------------------------------->

Im not going to dwell on this too much but the last bit of the process is done pretty much entirely in pandas (python data science library) and im trying some new organization below which im not 100% sold on. 
Basically the section titled "_Exporting Processed Data_" adds a couple new columns to the data, cleans up some loose ends, and ships the data as a csv in a form that is more polished than the pickle outputs.  
