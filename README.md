Scraping (with Ruby)
====================

A guide to scraping with Ruby, with a heavy emphasis on strategies.

We'll talk through the basics of scraping with Ruby, with a focus on the kinds of sites journalists most often encounter (read: .NET). So, we'll start with when to use which tools, including a full-blown browser emulator (Watir), a headless emulator (Mechanize) and an HTML parser (Nokogiri).


What we'll cover
----------------

First, we'll cover how to use a browser emulator (Selenium using Watir) to navigate a website and acquire the data-as-HTML that we are interested in.

Next, we'll demonstrate how to parse that HTML to extract the data that we are interested in. We'll just touch on spitting that information out to a CSV, as well as saving HTML to files. Long-term, you'll probably want to use an ORM (Active Record) and save records to a database.

Next, we'll talk about pitfalls to fret over when scraping sites, specifically .NET sites.

Finally, I'll offer my general advice for how scraping.


Why scrape (and what?)
----------------------

Alright, so there's a couple of good use cases for scraping. For me, the best use cases are:

1. When a records request is taking too long, costing too much or being refused, and the data is available online

2. When you want as real-time data as possible, making re-requests prohibitive

And the things that I am going for when I scrape are usually structured data held in tables and documents available as files.

General principles
------------------

1. Reduce moving parts by breaking up scraping tasks into separate programs
    * In Practice: scrape a list of pages with one scraper, scrape the actual pages with another
    * Reason: if something goes wrong, it's easier to diagnose and fix without starting all over again
2. Hit their server with as few requests as you can while accomplishing your goal
    * In Practice: though you want only some information from a page, save the entire page
    * Reasoning: storage is cheap, HTTP transfers take time, you don't want to trigger security measures, and you may discover anomalies you weren't anticipating when you explored a few pages


Setup for this session
----------------------

```bash
  $ mkdir ruby_scraping_course
  $ cd ruby_scraping_course
  $ mkdir profiles
  $ mkdir documents
```

The basics: getting and parsing HTML
====================================

Navigating websites with Watir
------------------------------

Watir is browser emulator that you can use to programmatically navigate websites. Long-term, I think it's best not to use Watir (more later), but for now, let's see how we can get around the web with it.

Next, let's open up `irb`, a Ruby interpreter that will let us execute Ruby commands:

```bash
    $ irb
```

For those not terribly familiar with Ruby, you can load libraries called gems that offer the useful functions that we'll employ in our code. For now, let's start with Watir-webdriver.

```ruby
    2.2.1 :001 > require 'watir-webdriver'
```

Alright, so now we can access all sorts of functions that will help us navigate the web with a friendly browser GUI.

Next, we have to open up a new browser instance. By default, Watir uses Firefox.

```ruby
    2.2.1 :001 > browser = Watir::Browser.new
```

This will open up a new Firefox window on your screen, and also create an object called `browser` that we can use to control the Firefox window.

Now, let's open up a website. For this example, I'm going to show you how to scrape data off of the state of Michigan's licensing site.

```ruby
    2.2.1 :001 > browser.goto 'https://w2.lara.state.mi.us/VAL/License/Search'
```

You should see the Firefox window load the website. Behind the scenes, our `browser` object also has access to the HTML of the site that our Firefox window has loaded, and we are going to use that to do a simple search of the site.

But first, we have to understand the most important tool in scraping: Firefox's (or Chrome's) Developer Tools.

Inspecting HTML with Developer Tools
------------------------------------

Starting from the assumption that we know how to get to the data that we want, we need to see how the exact language that the site we are visiting uses to retrieve information for us. In this case, that means finding out what those search fields that we want to use are actually called.

So let's bring up our Firefox window and we're going to right-click on the white text field to the right of `Last Name:` and select `Inspect Element (Q)`.

Alright, so what we are looking for is some identifying feature within this HTML that we can use to grab it using our `browser`. In this instance, we can see that the field is IDed as `LastName`, and we can use that ID to interact with it using the `browser`.

To start, let's drop in some text and see what we get back. "A" seems as good a place to start as any, so let's enter in an "A".

```ruby
    2.2.1 :001 > browser.text_field(id: 'LastName').set "A"
```

And after we hit enter, let's switch over to our Firefox window and verify that it popped an "A" into the box next to `Last Name:`.

So far so good, now we have to search. To do that, we'll need to get our `browser` to click that button at the bottom labeled `Search`.

Let's once again right-click on our target (the button) in Firefox and select `Inspect Element (Q)` to find out how we can ID it for the `browser`.

Ok, it's got an ID of `searchButton`, which we can use to click it.

```ruby
    2.2.1 :001 > browser.button({id: 'searchButton'}).click
```

And let's wait a minute while the server(s) in Michigan flip out with their highest simultaneous usage ever.

... ok, so now we have some results at the bottom responsive to our query.

The last essential thing for navigating the site is to be able to move around the result pages so you can pull in all of the records. Let's give that a shot by using the `browser`.

This page is simple enough: it uses a "Next" button to move forward. So, we find the ID of that button (nextPageButton) by right-clicking on it and using `Inspect Element`.

Now, we can click on it, just like we did with the search button.

```ruby
    2.2.1 :001 > browser.button({id: 'nextPageButton'}).click
```

This also gives us a convenient way to loop through the pages by checking to see if the Next button exists.

```ruby
    if browser.button({id: 'nextPageButton'}).exists?
        browser.button({id: 'nextPageButton'}).click
        #code for scraping results (below)
    end
```

For more on how you can interact with various web elements with Watir, click [here](http://watirwebdriver.com/web-elements/).

Parsing HTML with Nokogiri
--------------------------

Let's move on briefly to actually parsing the data on the page into a form we can save and use later. To do that, we'll use Nokogiri, which helps us access the HTML by taking advantage of the styling of the page.

```ruby
    2.2.1 :001 > require 'nokogiri'
```

Step one is to create a Nokogiri object out of the HTML stored in the `browser` object. We can do this with the following command:

```ruby
    2.2.1 :001 > page = Nokogiri::HTML(browser.html)
```

This command spits out the HTML from the `browser` (browser.html), and Nokogiri picks it up and creates a Nokogiri object called `page` that we can now parse with (relative) ease.

Nokogiri allows us to use the CSS selectors in the HTML to get to the portion of the HTML that we are interested in. In our case, we are interested in the table of records responsive to our request.

Once again, let's use the `Inspect Element (Q)` feature of Developer Tools to see what the HTML surround our table looks like.

Looking at the HTML structure in which the table is embedded, it looks like the lowest level we can get to and get the entire table is of the type `tbody`. Using Nokogiri, we can pull out a list of all `tbodies` in the HTML with the following command

```ruby
    2.2.1 :001 > tbodies = page.css('tbody')
```

This command searches the HTML stored in `page` for all `tbody` objects and creates a list out of them.

Next, we need to make sure we are getting the `tbody` we want. Let's start by seeing how many `tbodies` we got.

```ruby
    2.2.1 :001 > tbodies.length
```

You should get back 3. To figure out which one is the one we want, let's look at the text of each of them.

```ruby
    2.2.1 :001 > tbodies[0].text
```

This command returns the text within the HTML of the first `tbody` (index numbers start at 0). And this does not appear to be the one we want. Let's check the next one.

```ruby
    2.2.1 :001 > tbodies[1].text
```

Bingo.

Now that we've got the table, let's get the rows out of it into a new list that we can then pipe out to a csv. We can do that with Nokogiri, too.

```ruby
    2.2.1 :001 > rows = tbodies[1].css('tr')
```

Just as we could use the .css method to search our `page` object, we can use it to search out `tbodies` objects (they are of the same kind, one is just more narrow in terms of what HTML it encompasses). `tr`s are table rows.

Almost there. The next step is to get the data out into a csv.

(PS: For much more, check out [Dan Nguyen's Bastards of Ruby section on HTML parsing](http://ruby.bastardsbook.com/chapters/html-parsing/))

Saving to a csv
---------------

This isn't going to go in depth, just give you the basics. We'll use the "csv" gem to open a file and write out to it. In practice, use a database and Active Record.

First, we import the gem and create a new csv file with headers from the data.

```ruby
    require 'csv'
    headers = ['name','profession','license_type','license_number','licensee_url']
    csv_name='mi_licensees.csv'
    CSV.open(csv_name,"w") do |csv|
        csv << headers
    end
```

The headers you can see from the table, plus a field for the URL of each profile, which we'll make use of later.

Next, we'll loop over the rows of our table, which we grabbed above, and write them out to csv with the following code.

```ruby
    for row in rows
      cols = row.css('td')
      licensee_attributes = []
      for col in cols
        licensee_attributes.push(col.text)
      end
      
      licensee_attributes.push(row.css('a')[0]['href'])

      CSV.open(csv_name,"a") do |csv|
        csv << licensee_attributes
      end
    end
```

The first bit takes the text of each column and adds it to a list called `licensee_attributes`, which we then write out to our csv. Note the "a" rather than the "w" in the `CSV.open` command, which specifies that you want to append rather than overwrite.


Downloading the details pages
-----------------------------

Ok, now that we have links to each licensee's homepage, we can loop over them and make a copy of each page.

To do this, we are going to use Mechanize (in fact, we could have used Mechanize all along-and should-but Watir is easier to visualize). Mechanize will allow us to download each page, which we'll write out to a file.

```ruby
    require 'mechanize'
    mech_browser = Mechanize.new()
    mech_browser.verify_mode = OpenSSL::SSL::VERIFY_NONE #deals with SSL issue that I get from this particular server
    mech_browser.ssl_version = 'TLSv1'                   #deals with SSL issue that I get from this particular server
    mech_browser.user_agent_alias = 'Linux Firefox'      #I've run into several servers that seem to throw issues when not encountering a recognized user agent
```

Let's start by reading in our csv file.

```ruby
    2.2.1 :001 > licensees = CSV.read('mi_licensees.csv', headers:true)
```
Now we can use the address field we saved to open up each profile page and write it out to a text file.

```ruby
    url_prefix = 'https://w2.lara.state.mi.us'
    for licensee in licensees
        licensee_url = licensee['licensee_url']
        profile_page = mech_browser.get(url_prefix+licensee_url)
        File.open("profiles/"+licensee_url.split("/").last+".html","w") do |outfile|
            outfile.write(profile_page.body)
        end
    end
```

Downloading PDFs
----------------

A little detour for a semi-common scraping task: mass acquisition of PDFs.

To begin, I've looked up a profile that has disciplinary actions associated with it: license number 4301033666. That profile is [here](https://w2.lara.state.mi.us/VAL/License/Details/149912).

As you can see, Dr. Utarnachitt has got three documents online. Let's download these, using two different approaches: Mechanize and Watir.

With Mechanize, which I prefer when it is easily applied, you would grab a link and a PDF like so:

```ruby
    profile_page = mech_browser.get("https://w2.lara.state.mi.us/VAL/License/Details/149912")
    page = Nokogiri::HTML(profile_page.body)
    links = page.css('a') # getting the links from the css
    
    #getting just the document links
    pdf_links = []
    for link in links
        if link.text.include?('Download')
            pdf_links.push(link['href'])
        end
    end
    
    #now, just downloading one document
    pdf_link = pdf_links[0]
    document = mech_browser.get("https://w2.lara.state.mi.us/"+pdf_link)
    File.open("documents/"+pdf_link.split("/").last+".pdf","w") do |outfile|
        outfile.write(document.body)
    end
```

To do this with Watir, you need to first set up the browser so it won't ask you to download links that you click, like so:

```ruby
    profile=Selenium::WebDriver::Firefox::Profile.new
    profile['browser.download.folderList'] = 2 # custom location
    profile['browser.download.dir'] = Dir.pwd+"/documents"
    profile['browser.helperApps.neverAsk.saveToDisk'] = "text/csv,application/pdf,application/octet-stream,text/plain"
    profile['pdfjs.disabled'] = true
    profile['pdfjs.firstRun'] = false

    browser= Watir::Browser.new :firefox, :profile => profile
```

Then, we can load the page and click the link like so:

```ruby
    browser.goto ("https://w2.lara.state.mi.us/VAL/License/Details/149912")
    link = browser.link :text => 'Download' #note that this will grab the first one meeting this criterion: loop using browser.goto and feed it URLs if you need all the documents
    link.click
```

I'm sure you can tweak the settings somehow, but this offers a less ideal solution than Mechanize for several reasons. One major one is that I could choose the name of the downloaded file explicitly to avoid possible overwriting (because yes, government agencies do sometimes post different PDFs with the exact same name).

    

Pitfalls
========

Suspicious record/page counts
-----------------------------

Let's take a look again at our the results in our Firefox browser for a search on the letter 'A'. At the bottom of the first page of results, it shows us that we have exactly 10 pages.

10 seems a little round. Let's try searching for B (for these purposes, I'll just type it into the browser and click the button).

Hmmm. 10 pages again. That seems implausible, given that those with last names beginning with B outnumber those beginning with an A nationwide by about 2.5 times, according to the Census.

In this case, you would need to write a function that adds a letter if the page count came in at less than 10. If not, add a letter and try again. If so, scrape and move up the alphabet by a letter.

CODE SNIPPET

Duplicate records
-----------------

With .NET sites and Watir, I have found (without determining the exact cause) that scrapers using Watir can sometimes get ahead of themselves, and end up re-writing the same results page. If this happens, you'll end up with one page of licensees being repeated, and another being omitted.

To prevent this, you have two solutions: write a function to check the first N names of the table against the first N names of the table on the previous page. That solution is kinda ugly.

Another, more elegant solution, is to avoid using Watir altogether and to instead use Mechanize for the entire process. I'll put up an example of this, because it's a bit more convoluted, but definitely the way to go if you're embarking on a long project.


Now for some general advice
===========================

Reverse engineering sites
-------------------------

Scraping a website is 10% writing code to solve problems, 30% figuring out how to structure what you pull down and 60% reverse engineering the site.  One really nice way to do that that I haven't touched on, but which will come up is examining the requests that your browser is sending. You can find this by opening the Network Monitor tab of developer tools.

Here, you can get the real details of what information your browser is exchanging with their server to get you the information you want. You can see if requests are being sent with cookies stored in the browser. If the site relys on Javascript links, you can see what information is being sent when you click the links.

The best way to scrape, if you get comfortable with it, is to breakdown the structure of these requests and find some programmatic way to submit them, cutting out emulators like Watir.

Structuring your scrapers and your database
-------------------------------------------

Ideally, what I propose one do is follow a general procedure:

1. Carefully imagine what the database you want to end up with afterwards is going to look like
    * I vote that you strongly consider the Active Record/Django/sane approach, wherein you create separate tables with foreign keys linking back to your base unit (in this case, a licensee table could be linked one-to-many to a documents table)
    * I highly recommend storing everything in a relational database rather than CSVs (again, Active Record can help you here)
    * I also suggest finding an example of what you're looking for through other means so you can see what the page will look like (in this case, a licensee with a disciplinary document could be looked up in media accounts or regulator board meeting minutes)
2. Grab your list(s) first, their details later
    * Once you've settled on a database structure, ideally fill it using discrete scrapers, rather than one all at once
    * For instance, if you set up a table of licensees, fill it with the fields that show up in the search results in one run, and then...
3. Grab the HTML of the details/profiles, etc.
    * For this site, I'd create a "text" field in MySQL for the licensee table, and place the HTML of the page in there for later inspection
4. Grab the data you want out of what you got in step 2
    * Now that I've got the site essentially copied, I can just yank data from the profiles I am interested in (let's say, again, those with disciplinary information) using a SQL query
    * So I can now create a table of documents that includes the file name, document type and document URL
    * Finally, I can scrape those documents with a program that just loops over the URLs and saves files

Why do I prefer to do it this way?


1. Because if something breaks, it is much easier for me to figure out what went wrong
2. Because if it does go wrong, it's easier to restart it without starting over
3. Because if I mess up, I don't have to hit their servers again to fix it
4. Because by thinking about it this way, I end up with a very well-organized database of the site
5. Because anything you can do to stop using a visual browser (Watir) is going to speed up the process


Finally, some random, semi-common obstacles to this approach
============================================================

For a number of sites, this approach isn't going to work. Here are a number of things that could happen to stop you from doing this, and how you can get around them.

Postbacks/Javascript links
--------------------------

Sometimes, links aren't going to be absolute, like they were in Michigan. In this case, you have two options. The simplest is to click the link with Watir and save the HTML directly to a file, then send a back command, and move on.

Better than this, probably, is to deconstruct the POST request that is being submitted when you click the button (using the Network tab in Developer Tools), and use Mechanize to download the HTML.


Sites using cookies/requiring a certain user-agent
--------------------------------------------------

Some sites have absolute links, but they require your browser to have cookies to carryout your request. The solution here is to see what cookies are included in your GET request. You can then, usually, just pass a value derived from examining a GET request into your Mechanize browser.

Similar things can come up with User-Agents, which is why I always asign one when I initiate a Mechanize browser (see above).

Sites that generate temporary links
-----------------------------------

I've only run into this once. At this point, my general theory breaks down: you're pretty much going to have to pull everything all at once, rather than separating the scraper into discrete parts.


Sites that use CAPTCHAs
-----------------------

So you're interested in the dark arts? Inquire within.

Questions?
==========

Ask John. I prefer Python.
