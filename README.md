# Pulitzer Prize Winners - Book Categories

I volunteer at my kids' school library.  I recently needed a list of all the Pulitzer Prize winners, in order to identify which books we had in our catalogue but weren't on our Pulitzer winners bookshelf.  I had resigned myself to manually copy-pasting almost six hundred entries in eight categories from the Pulitzer website - a task that would likely take the better part of a month - before then going through the process of checking our library catalogue for matches, but one midnight I had an ephiphany that there might have been another person with the same needs that had already created a database.

I found one better: https://github.com/jonseitz/pulitzer-scraper had identified nine years ago that the Pulitzer website itself uses API calls to inform its category pages!

## API Call URLs
https://github.com/jonseitz/pulitzer-scraper had identified that the Pulitzer website makes the browser make API calls for JSON objects.  I was able to confirm this with Developer Tools: the same URL - save for HTTPS vs HTTP - e.g. `https://www.pulitzer.org/cms/api/1/winners/cat/[category ID]/raw.json`.

However, it appears the server (i.e. the Cloudflare CDN) doesn't like the `curl` client:

```
[terry@llnw3 ~]$ curl 'https://www.pulitzer.org/cms/api/1/winners/cat/219/raw.json'
<!DOCTYPE html><html lang="en-US"><head><title>Just a moment...</title><SNIP>[terry@llnw3 ~]$ 
[terry@llnw3 ~]$
```

The `user-agent:` and `cookie:` request headers from the browser were enough to satisfy this requirement.

## Book Category IDs
The [Pulitzer Prize Winners by Category](https://www.pulitzer.org/prize-winners-categories) page links to the categories.  I only needed the book categories, so the categories in the section "Letters, Drama & Music" except for the "Music" category.

* Fiction: 219
* Drama: 218
* History: 220
* Biography: 222
* Memoir or Autobiography: 650
* Poetry: 224
* General Nonfiction: 223
* Novel: 261

## Collect Data
Let's grab all the raw JSON to fiddle with them locally:
```
[terry@llnw3 Downloads]$ mkdir pulitzer; cd $_
[terry@llnw3 pulitzer]$ 
[terry@llnw3 pulitzer]$ 
[terry@llnw3 pulitzer]$ for category in 219 218 220 222 650 224 223 261; do echo "----- $category -----"; curl -s "https://www.pulitzer.org/cms/api/1/winners/cat/$category/raw.json" -H 'cookie: <SNIP>' -H 'user-agent: <SNIP>' | jq > $category.json; done
----- 219 -----
----- 218 -----
----- 220 -----
----- 222 -----
----- 650 -----
----- 224 -----
----- 223 -----
----- 261 -----
[terry@llnw3 pulitzer]$ 
[terry@llnw3 pulitzer]$ 
[terry@llnw3 pulitzer]$ 
[terry@llnw3 pulitzer]$ wc -l *
    3393 218.json
    3092 219.json
    4407 220.json
    4490 222.json
    2899 223.json
    4364 224.json
    1028 261.json
      90 650.json
   23763 total
[terry@llnw3 pulitzer]$ 
```

## Schema
The first winner in the Fiction category:
```
  {
    "title": "Jayne Anne Phillips",
    "nid": "22533",
    "type": "winner",
    "changed": "1730230229",
    "field_category": {
      "und": [
        {
          "tid": "219"
        }
      ]
    },
    "field_year": {
      "und": [
        {
          "tid": "651"
        }
      ]
    },
    "field_abbr_citation": {
      "und": [
        {
          "safe_value": "A beautifully rendered novel set in West Virginia’s Trans-Allegheny Lunatic Asylum in the aftermath of the Civil War where a severely wounded Union veteran, a 12-year-old girl and her mother, long abused by a Confederate soldier, struggle to heal."
        }
      ]
    },
    "field_location_text": [],
    "field_publication": {
      "und": [
        {
          "safe_value": "Night Watch"
        }
      ]
    },
    "field_publisher": {
      "und": [
        {
          "safe_value": "Knopf"
        }
      ]
    },
    "field_teaser_thumbnail": [],
    "path_alias": "winners/jayne-anne-phillips"
  },
```
Eyeballing this yields the following schema:
* `.title`: counterintuitively, the author (not the book title)
* `.nid`: ?
* `.type`: winner, as opposed to “Nominated” (this is the API output for winners)
* `.changed`: turns out this is either “created” or “last modified”, not the actual award year
* `.field_category.und[0].tid`: the category - 219 is “Fiction”
* `.field_year.und[0].tid`: the award year, apparently, although still no idea how to grok the six-hundred series
* `.field_abbr_citation.und[0].safe_value`: looks like the description
* `.field_location_text`: ???
* `.field_publication.und[0].safe_value`: the book title.  What is “safe_value”?
* `.field_publisher.und[0].safe_value`: publisher?
* `.field_teaser_thumbnail`: ?
* `.path_alias`: does this map to a URL? Looks like yes https://www.pulitzer.org/winners/jayne-anne-phillips

## jq
`jq` is my go-to tool to handle JSON inputs to make human-readable output.  We're working with JSON here - simple enough?  But no, it dies fourteen lines in:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(.title)\t\(.field_publication.und[].safe_value)"'
Jayne Anne Phillips     Night Watch
Barbara Kingsolver      Demon Copperhead
Hernan Diaz     Trust
Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
Louise Erdrich  The Night Watchman
Colson Whitehead        The Nickel Boys
Richard Powers  The Overstory
Andrew Sean Greer       Less
Colson Whitehead        The Underground Railroad
Viet Thanh Nguyen       The Sympathizer
Anthony Doerr   All the Light We Cannot See
Donna Tartt     The Goldfinch
Adam Johnson    The Orphan Master&#039;s Son
jq: error (at <stdin>:3092): Cannot index array with string "und"
[terry@llnw3 pulitzer]$ 
```

The year before *The Orphan Master's Son* was the year no award was given: https://www.pulitzer.org/prize-winners-by-category/219

### `jq` dying over "No award" years
Ah, the JSON entry for the "No award" year appears to have no `.field_publication.und[0].safe_value`:
```
  {
    "title": "No award",
    "nid": "7149",
    "type": "winner",
    "changed": "1439235215",
    "field_category": {
      "und": [
        {
          "tid": "219"
        }
      ]
    },
    "field_year": {
      "und": [
        {
          "tid": "108"
        }
      ]
    },
    "field_abbr_citation": {
      "und": [
        {
          "safe_value": "No award\n"
        }
      ]
    },
    "field_location_text": [],
    "field_publication": [],
    "field_publisher": [],
    "field_teaser_thumbnail": [],
    "path_alias": "winners/no-award-4"
  },
```

I tried a bunch of things with `select()` and `try()` and `[key]?`, but nope: https://stackoverflow.com/questions/42097410/how-to-check-for-presence-of-key-in-jq-before-iterating-over-the-values

I had switched to [Warp](https://www.warp.dev/) at this time because Warp allows you to click into the middle of your command to edit it (like a text editor), instead of having to scrub using the arrow keys (or Ctrl+arrow to jump word-by-word).  Turns out Warp also has an AI agent!  Asked it for help, and:
> Now I can see the problem. There are entries where field_publication is an empty array ([]) rather than an object with an "und" property. We need to modify our jq command to handle this case by checking if field_publication is both non-null AND an object. Here's the fixed command:
> ```
> cat 219.json | jq -r '.[] | select(.field_publication != null and (.field_publication | type == "object")) | .field_publication.und[0].safe_value'
> ```

(Whoops, I did not notice that `field_publication` itself does exist in the "No award" object.)

This, however, still makes `jq` skip the "No award" lines themselves, because `.field_publication` is empty:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(.title)\t\(select(.field_publication[] != null) | .field_publication.und[0].safe_value)"' | head -15 
Jayne Anne Phillips     Night Watch
Barbara Kingsolver      Demon Copperhead
Hernan Diaz     Trust
Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
Louise Erdrich  The Night Watchman
Colson Whitehead        The Nickel Boys
Richard Powers  The Overstory
Andrew Sean Greer       Less
Colson Whitehead        The Underground Railroad
Viet Thanh Nguyen       The Sympathizer
Anthony Doerr   All the Light We Cannot See
Donna Tartt     The Goldfinch
Adam Johnson    The Orphan Master&#039;s Son
Jennifer Egan   A Visit from the Goon Squad
Paul Harding    Tinkers
[terry@llnw3 pulitzer]$ 
```

Warp's AI to the rescue!
> The command is now working correctly! It's displaying both the title and publication for all entries, using the title itself when there's no publication value. The output shows all entries, including the "No award" and "No award given" entries, with their titles properly displayed.
> 
> Here's what the command does:
> 1. For each entry (.[]): prints the title
> 2. Adds a tab character (\t)
> 3. If field_publication exists and is an object, prints the publication value
> 4. Otherwise, falls back to printing the title
> ```
> cat 219.json | jq -r '.[] | "\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value else .title end)"'
> ```

`else .title` makes it print the `.title` twice, resulting in "No award No award":
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value else .title end)"' | head -15
Jayne Anne Phillips     Night Watch
Barbara Kingsolver      Demon Copperhead
Hernan Diaz     Trust
Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
Louise Erdrich  The Night Watchman
Colson Whitehead        The Nickel Boys
Richard Powers  The Overstory
Andrew Sean Greer       Less
Colson Whitehead        The Underground Railroad
Viet Thanh Nguyen       The Sympathizer
Anthony Doerr   All the Light We Cannot See
Donna Tartt     The Goldfinch
Adam Johnson    The Orphan Master&#039;s Son
No award        No award
Jennifer Egan   A Visit from the Goon Squad
[terry@llnw3 pulitzer]$ 
```

So let's make it `else ""`:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value else "" end)"' | head -15
Jayne Anne Phillips     Night Watch
Barbara Kingsolver      Demon Copperhead
Hernan Diaz     Trust
Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
Louise Erdrich  The Night Watchman
Colson Whitehead        The Nickel Boys
Richard Powers  The Overstory
Andrew Sean Greer       Less
Colson Whitehead        The Underground Railroad
Viet Thanh Nguyen       The Sympathizer
Anthony Doerr   All the Light We Cannot See
Donna Tartt     The Goldfinch
Adam Johnson    The Orphan Master&#039;s Son
No award
Jennifer Egan   A Visit from the Goon Squad
[terry@llnw3 pulitzer]$ 
```

### `\n` in older book titles
For some reason, some of the earlier entries have a rogue newline character at the end of the book title:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq '.[] | (if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value else "" end)' | tail
"The Travels of Jaimie McPheeters\n"
"A Death In The Family\n"
"Andersonville\n"
"A Fable\n"
"The Old Man and the Sea\n"
"The Caine Mutiny\n"
"The Town\n"
"The Way West\n"
"Guard of Honor\n"
"Tales of the South Pacific\n"
[terry@llnw3 pulitzer]$ 
```

A `jq` incantation to make the special characters go away: https://stackoverflow.com/questions/40321035/remove-escape-sequence-characters-like-newline-tab-and-carriage-return-from-jso/40321859#40321859

Pipe the book titles into it:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq '.[] | (if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)' | tail
"The Travels of Jaimie McPheeters"
"A Death In The Family"
"Andersonville"
"A Fable"
"The Old Man and the Sea"
"The Caine Mutiny"
"The Town"
"The Way West"
"Guard of Honor"
"Tales of the South Pacific"
[terry@llnw3 pulitzer]$ 
```

### Year Numbering Scheme
https://github.com/jonseitz/pulitzer-scraper already identified that the years "105" to "203" map to 2015 and 1917 respectively: https://github.com/jonseitz/pulitzer-scraper/blob/master/README.md?plain=1#L85C1-L85C74
> The `tid` range for years goes from **105** _(2015)_ - **203** _(1917)_. 

There's also a "104", which is likely 2016.  But!  A bunch of numbers in the *six hundred* range after 104:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(.field_year.und[0].tid)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' | head -15
651     Jayne Anne Phillips     Night Watch
645     Barbara Kingsolver      Demon Copperhead
645     Hernan Diaz     Trust
640     Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
632     Louise Erdrich  The Night Watchman
627     Colson Whitehead        The Nickel Boys
621     Richard Powers  The Overstory
613     Andrew Sean Greer       Less
601     Colson Whitehead        The Underground Railroad
104     Viet Thanh Nguyen       The Sympathizer
105     Anthony Doerr   All the Light We Cannot See
106     Donna Tartt     The Goldfinch
107     Adam Johnson    The Orphan Master&#039;s Son
108     No award
109     Jennifer Egan   A Visit from the Goon Squad
[terry@llnw3 pulitzer]$ 
```

AI was already a great help, let's keep asking it for help:
> To derive a formula that converts the numbers ("651", "645", etc.) to the corresponding years, we need to identify the relationship between the input and output.
> > 
> Let:
> * $x$ represent the input number.
> * $y$ represent the corresponding year.
> 
> From the data, we observe:
> 
> * $x = 651, y = 2024$
> * $x = 645, y = 2023$
> * $x = 640, y = 2022$
> * $x = 632, y = 2021$
> * $x = 627, y = 2020$
> * $x = 621, y = 2019$
> * $x = 613, y = 2018$
> * $x = 601, y = 2017$
> 
> The goal is to find $y = f(x)$.
> 
> **Step 1: Identify the pattern**
> 
> The differences between consecutive $x$ values are irregular:
> 
> * $651-645 = 6$
> * $645-640 = 5$
> * $640-632 = 8$
> * $632-627 = 5$
> * $627-621 = 6$
> * $621-613 = 8$
> * $613-601 = 12$
> 
> The differences in $y$ values are constant ($-1$).
> 
> **Step 2: Establish a linear relationship**
> 
> Let $y=mx+b$. Solve for $m$ and $b$ using two points:
> 
> 1. $(651,2024)$
> 2. $(601,2017)$
> 
> From the linear equation:
> 
> $$
> m = \frac{y_2 - y_1}{x_2 - x_1} = \frac{2017 - 2024}{601 - 651} = \frac{-7}{-50} = \frac{7}{50}
> $$
> 
> Substitute $m = \frac{7}{50}$ into $y = mx + b$ and use $(651, 2024)$:
> 
> $$
> 2024 = \frac{7}{50}(651) + b
> $$
> 
> $$
> 2024 = \frac{4557}{50} + b
> $$
> 
> $$
> b = 2024 - 91.14 = 1932.86
> $$
> 
> Thus, the formula is:
> 
> $$
> y = \frac{7}{50}x + 1932.86
> $$
> 
> **Step 3: Verify the formula**
> 
> For $x = 651$:
> 
> $$
> y = \frac{7}{50}(651) + 1932.86 = 2024
> $$
> 
> For $x = 601$:
> 
> $$
> y = \frac{7}{50}(601) + 1932.86 = 2017
> $$
> 
> The formula satisfies all the given conversions.

"The formula satisfies all the given conversions." ... Really?  Kinda sorta:
```
[terry@llnw3 pulitzer]$ echo -e "601 2017\n613 2018\n621 2019\n627 2020\n632 2021\n640 2022\n645 2023\n651 2024" | awk '{print $1,$2,((7/50)*$1 + 1932.86)}'
601 2017 2017
613 2018 2018.68
621 2019 2019.8
627 2020 2020.64
632 2021 2021.34
640 2022 2022.46
645 2023 2023.16
651 2024 2024
[terry@llnw3 pulitzer]$ 
```

Looks like ChatGPT fell into the fallacy of whatever-it's-called, where it only tested two things and assumed correctness for the untested others.  ChatGPT offered to try to come up with a better-matching model -
> If a stricter match is needed, a piecewise or non-linear formula might be more approprirate.  Let me know if you'd like me to refine the model further.

I've got a better idea: maybe whoever designed this numbering scheme also decided to drop the decimals.

ChatGPT, what might the number for 2025 be?
> For $y = 2025$, the corresponding $x$ value would be approximately $658.14$.

```
[terry@llnw3 pulitzer]$ echo "658 2025?" | awk '{print $1,$2,((7/50)*$1 + 1932.86)}'
658 2025? 2024.98
[terry@llnw3 pulitzer]$ echo "658.14 2025?" | awk '{print $1,$2,((7/50)*$1 + 1932.86)}'
658.14 2025? 2025
[terry@llnw3 pulitzer]$ echo "659 2025?" | awk '{print $1,$2,((7/50)*$1 + 1932.86)}'
659 2025? 2025.12
[terry@llnw3 pulitzer]$ 
```

I guess let's find out what number the Pulitzer website ends up using, when the 2025 winners are posted.  Until then, let's take ChatGPT at its word: an `if else then` statement to calculate the two different types of years (need to use `|tonumber` because, for some reason, `.field_year.und[0].tid` are strings)
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86) end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' | head -15
2024    Jayne Anne Phillips     Night Watch
2023.1599999999999      Barbara Kingsolver      Demon Copperhead
2023.1599999999999      Hernan Diaz     Trust
2022.4599999999998      Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
2021.34 Louise Erdrich  The Night Watchman
2020.6399999999999      Colson Whitehead        The Nickel Boys
2019.8  Richard Powers  The Overstory
2018.6799999999998      Andrew Sean Greer       Less
2017    Colson Whitehead        The Underground Railroad
2016    Viet Thanh Nguyen       The Sympathizer
2015    Anthony Doerr   All the Light We Cannot See
2014    Donna Tartt     The Goldfinch
2013    Adam Johnson    The Orphan Master&#039;s Son
2012    No award
2011    Jennifer Egan   A Visit from the Goon Squad
[terry@llnw3 pulitzer]$ 
```

`|floor` to drop the decimal:
```
[terry@llnw3 pulitzer]$ cat 219.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' | head -15
2024    Jayne Anne Phillips     Night Watch
2023    Barbara Kingsolver      Demon Copperhead
2023    Hernan Diaz     Trust
2022    Joshua Cohen    The Netanyahus: An Account of a Minor and Ultimately Even Negligible Episode in the History of a Very Famous Family
2021    Louise Erdrich  The Night Watchman
2020    Colson Whitehead        The Nickel Boys
2019    Richard Powers  The Overstory
2018    Andrew Sean Greer       Less
2017    Colson Whitehead        The Underground Railroad
2016    Viet Thanh Nguyen       The Sympathizer
2015    Anthony Doerr   All the Light We Cannot See
2014    Donna Tartt     The Goldfinch
2013    Adam Johnson    The Orphan Master&#039;s Son
2012    No award
2011    Jennifer Egan   A Visit from the Goon Squad
[terry@llnw3 pulitzer]$ 
```

## Export
I like using tab-separated values rather than comma-separated values, because rarely do entries have tab characters whereas any quoted commas in a CSV will render `cut -d, -f [fields]` useless.  Print the column headers and then export the jq output to a TSV file:
```
[terry@llnw3 pulitzer]$ workingfile=(219 fiction); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(218 drama); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(220 history); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(222 biography); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(650 memoir); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(224 poetry); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(223 generalnonfiction); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ workingfile=(261 novel); echo -e "year\tauthor\ttitle" > export_${workingfile[1]}.tsv; cat ${workingfile[0]}.json | jq -r '.[] | "\(if .field_year.und[0].tid|tonumber < 204 then (2016 - ((.field_year.und[0].tid|tonumber) - 104)) else (((7/50) * (.field_year.und[0].tid|tonumber)) + 1932.86)|floor end)\t\(.title)\t\(if .field_publication and (.field_publication | type == "object") then .field_publication.und[0].safe_value | gsub("\\p{Cc}"; "") else "" end)"' >> export_${workingfile[1]}.tsv
[terry@llnw3 pulitzer]$ 
```

These files appear in this repository in the _data directory.
