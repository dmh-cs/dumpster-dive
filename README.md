# A whole Wikipedia, sitting in mongodb
put your hefty [wikipedia dump](https://dumps.wikimedia.org) into mongo, with fully-parsed wikiscript - without thinking, without loading it into memory, grepping, unzipping, or other crazy command-line nonsense.

It's a javascript one-liner that puts a highly-queryable wikipedia on your laptop in a nice afternoon.

It uses [wtf_wikipedia](https://github.com/spencermountain/wtf_wikipedia) to parse wikiscript into *almost-nice* json.

```bash
npm install -g wikipedia-to-mongodb
```
### ⚡ From the Command-Line:
```bash
wp2mongo /path/to/my-wikipedia-article-dump.xml.bz2
```
### 😎 From a nodejs script
```js
var wp2mongo = require('wikipedia-to-mongodb')
wp2mongo({file:'./enwiki-latest-pages-articles.xml.bz2', db: 'enwiki'}, callback)
```

then check out the articles in mongo:
````javascript
mongo        //enter the mongo shell
use enwiki   //grab the database

db.wikipedia.find({title:"Toronto"})[0].categories
//[ "Former colonial capitals in Canada",
//  "Populated places established in 1793",
//  ...]
db.wikipedia.count({type:"redirect"})
// 124,999...
````

### how it works:
this library uses:
* [unbzip2-stream](https://github.com/regular/unbzip2-stream) to stream-uncompress the gnarly bz2 file

* [xml-stream](https://github.com/assistunion/xml-stream) to stream-parse its xml format

* [wtf_wikipedia](https://github.com/spencermountain/wtf_wikipedia) to brute-parse the article wikiscript contents into JSON.

* [redis](http://redis.io/) to (optionally) put wikiscript parsing on separate threads :metal:

# 1)
you can do this.
a few Gb. you can do this.
you can do this.
# 2) get ready
Install [nodejs](https://nodejs.org/en/), [mongodb](https://docs.mongodb.com/manual/installation/), and optionally [redis](http://redis.io/)

```
git clone git@github.com:spencermountain/wikipedia-to-mongodb.git
cd wikipedia-to-mongodb
npm install
```

# 3) download wikipedia
The Afrikaans wikipedia (only 33 556 artikels) only takes a few minutes to download, and 10 mins to load into mongo on a macbook:
```bash
# dowload an xml dump (38mb, couple minutes)
wget https://dumps.wikimedia.org/afwiki/latest/afwiki-latest-pages-articles.xml.bz2
```
the english/german ones are bigger. Use whichever xml dump you'd like.

# 4) get going
```bash
#load it into mongo (10-15 minutes)
wikipedia-to-mongodb ./afwiki-latest-pages-articles.xml.bz2
```

# 5) check out your data
to view your data in the mongo console,
````javascript
$ mongo
use af_wikipedia

//shows a random page
db.wikipedia.find().skip(200).limit(2)

//count the redirects (~5,000 in afrikaans)
db.wikipedia.count({type:"redirect"})

//find a specific page
db.wikipedia.findOne({title:"Toronto"}).categories
````


## Same for the English wikipedia:
the english wikipedia will work under the same process, but
the download will take an afternoon, and the loading/parsing a couple hours. The en wikipedia dump is a 4gb download and becomes a pretty legit mongo collection uncompressed. It's something like 40gb, but mongo can do it... You can do it!


### Options
#### human-readable plaintext **--plaintext**
```js
wp2mongo({file:'./myfile.xml.bz2', db: 'enwiki', plaintext:true}, console.log)
/*
[{
  _id:'Toronto',
  title:'Toronto',
  plaintext:'Toronto is the most populous city in Canada and the provincial capital...'
}]
*/
```
#### go faster with Redis **--worker**
there is yet much faster way (even x10) to import all pages into mongodb but a little more complex. it requires redis installed on your computer and running worker in separate process.

It also gives you a cool dashboard, to watch the progress.
````bash
# install redis
sudo apt-get install # (or `brew install redis` on a mac)

# clone the repo
git clone git@github.com:spencermountain/wikipedia-to-mongodb.git && cd wikipedia-to-mongodb

#load pages into job queue
wp2mongo ./afwiki-latest-pages-articles.xml.bz2 --worker

# start processing jobs (parsing articles and saving to mongodb) on all CPU's
node src/worker.js

# you can preview processing jobs in kue dashboard (localhost:3000)
node node_modules/kue/bin/kue-dashboard -p 3000
````

### Addendum:
#### \_ids
since wikimedia makes all pages have globally unique titles, we also use them for the mongo `_id` fields.
The benefit is that if it crashes half-way through, or if you want to run it again, running this script repeatedly will not multiply your data. We do a 'upsert' on record.

#### encoding special characters
mongo has some opinions on special-characters in some of its data. It is weird, but we're using this [standard(ish)](https://stackoverflow.com/a/30254815/168877) form of encoding them:
```
\  -->  \\
$  -->  \u0024
.  -->  \u002e
```

MIT