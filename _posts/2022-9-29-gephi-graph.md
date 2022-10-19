---
title: Making a Social Graph with python & gephi
date: 2022-10-19 12:00:00 -500
categories: [technical writing]
tags: [python, gephi, graph]
---

# Prerequisites

You'll need the following to create this project.

* An instagram account.
* Python - Download [here](https://www.python.org/downloads/).
* Gephi - Download [here](https://gephi.org/users/download/).

The file structure of the project used is as followed,

./data/{account_scraped}/following.har  
./data/{account_scraped}/profile.json

./out/people.json  
./out/graph/edges.csv
./out/graph/nodes.csv

./src/extract.py  
./src/process.py

# Step 1 - Collecting the data

I used my fantasy football league as the sample. We all met in college, and some guys are from the same hometown, so it made for a good sample to see clusters overlap, as well as secluded ones. For the record our league is called the "League of Nations" and each team is a nation. So if you see that throughout don't fret.

## Go to an instagram account

When you have the profile you'd like to scrape in front of you, go ahead and open the dev tools. It should appear like below, and you are going to click on the network portion which is where the green arrow is pointing too.

[devtools](/assets/ig1.jpg)


## Collect and save followings of the profile

The network log will begin to record the activity as you scroll through the followings. Scroll all the way to the end of the accounts listed in following. 

Then save the activity log by clicking the button in the top right, see green arrow in screenshot.

[network_log](/assets/ig2.jpg)

Make sure you save the file as an HTTP archive (.har) as follows:

 ./data/{account_scraped}/(save here)

I saved all of these files as _following.har_.

## Save the profile JSON file

At the end of the url you are going to insert either ?__a=1 OR ?__a=1&__d=dis. If one does not work the other should. You should get the following.

[profile_json](/assets/profilejson.jpg){: w="700" h="400" }

Save this to ./data/{account_scraped}/(save here) as a JSON file.

Again, saved each as _profile.json_ under the acct_scraped folder within the data folder.

 ./data/{account_scraped}/(save here)

Make sure the files are complete so you do not have any issues extracting the data you'd like.

# Step 2 - Extracting the data

This is where python comes in. Go ahead and create folder named "src" and a file called "extract.py". It should be organized as below.

./src/extract.py

We are going to import the following:

{% highlight python %}
import json
import os
import re
import base64
{% endhighlight %}

The idea of this script is to extract the information we want so we can process it, and then visualize it. To start, let's get the followings of the user which we have in the http archive files (following.har).

## This function extracts the profile information of the user.

{% highlight python %}
def extract_profile(data) -> list:
    return data['graphql']['user']
{% endhighlight %}

## This function extracts the followings of the user.

{% highlight python %}
def extract_followings(data) -> list:
    friendship_api_url = r'https://i.instagram.com/api/v1/friendships/.*/following'

    contents = [entry['response']['content'] for entry in data['log']['entries']
                if re.match(friendship_api_url, entry['request']['url'])]

    # extract json data
    users = []
    for content in contents:
        if 'text' not in content:
            continue
        text = content['text']

        if 'encoding' in content:
            encoding = content['encoding']
            if encoding == 'base64':
                decodedBytes = base64.b64decode(text)
                decodedStr = str(decodedBytes, 'utf-8')
                user_json = json.loads(decodedStr)
                users.extend(user_json['users'])
        else:
            user_json= json.loads(text)
            users.extend(user_json['users'])
        
    return users
{% endhighlight %}

## This function extracts the profile and followings of the user.

{% highlight python %}
def extract_person(person: str):
    with open(f'./data/{person}/following.har', 'r', errors='ignore') as f:
        data = json.loads(f.read())
        followings = extract_followings(data)

    with open(f'./data/{person}/profile.json', 'r') as f:
        data = json.loads(f.read())
        profile = extract_profile(data)

    output = {
        'general': profile,
        'followings': followings
    }
    return output, profile['id']
{% endhighlight %}

## This function extracts the profile and followings of all the users. 

The file appears in ./out/people.json

{% highlight python %}
def extract_all():
    persons = os.listdir('./data')

    users = {}
    for person in persons:
        output, id = extract_person(person)
        users[id] = output

    with open('./out/people.json', 'w') as f:
        f.write(json.dumps(users))
{% endhighlight %}

## This function call extracts the profile and followings of all the users.

{% highlight python %}
extract_all()
{% endhighlight %}

# Step 3 - Processing the data 

Now that we have the data extracted, we need to process it into nodes and edges so we can easily visualize it with gephi. So go ahead and make a new python file in the src folder titled _process.py_

./src/process.py

Once again, the purpose of this script is to take the data we have extracted which can be found in ./out/people.json , but first we have to import the following.

{% highlight python %}
import json
import pandas as pd
{% endhighlight %}

## Open people.json

Once we've imported what we need, let us go ahead and open the _people.json_ file and load it into a variable called data.

{% highlight python %}
with open ('./out/people.json', 'r') as f:
    data = json.loads(f.read())
{% endhighlight %}

## Nodes and Edges list

After we have opened the _people.json_ file we are going to create two empty lists called 'nodes' and 'edges'.

{% highlight python %}
nodes = []
edges = []
{% endhighlight %}

## Loop through data and acct info for nodes

After our lists are created, we are going to loop through the data and add the account's information to the nodes list. In this example we wanted the id, username, full name and to know if the account was private or not.

{% highlight python %}
for person_id, person_data in data.items():
    general = person_data['general']
    nodes.append({
        'id': general ['id'],
        'label': general['username'],
        'full_name': general['full_name'],
        'is_private': general['is_private']
    })
{% endhighlight %}

## Loop through followings for nodes

After we have the desired account information we are going to loop through the followings of the account and add their information to the nodes list.

{% highlight python %}
    followings = person_data['followings']
    for follow in followings:
        nodes.append({
        'id': follow['pk'],
        'label': follow['username'],
        'full_name': follow['full_name'],
        'is_private': follow['is_private']
     })
{% endhighlight %}

## Loop through followings for edges

After we have the nodes list in order, we are going to now loop through the followings again, but this time we will be adding an edge to the edges list.

{% highlight python %}
    for follow in followings: 
        edges.append({
            'source': person_id,
            'target': follow['pk']
        })
{% endhighlight %}

## Convert nodes and edges list to pandas DataFrame and save as a csv file.

Now all we have to do is convert our lists and save them as a .csv file for gephi.

{% highlight python %}
nodes_df = pd.DataFrame.from_dict(nodes)
edges_df = pd.DataFrame.from_dict(edges)

nodes_df.to_csv('./out/graph/nodes.csv', index=False, header=True)
edges_df.to_csv('./out/graph/edges.csv', index=False, header=True)
{% endhighlight %}

Now all you have to do is open gephi, click on the data laboratory tab, then import your newly saved nodes.csv and edges.csv.

After, I'd play around with the different algorithms and see what each of them do, and get used to the customization features. It's an awesome visualization tool. Hopefully you learned a thing or two by reading this. Free of charge, as it should be.