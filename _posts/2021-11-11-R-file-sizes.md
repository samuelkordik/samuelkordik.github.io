---
layout: post
title: Interactive R for File System Stats
date: 2021-11-11 07:13:00
description: Using R in the shell to understand common file sizes
tags: R scripting
categories: R
---

I recently wrote guidelines for others to use when developing policy and procedure documents. Part of these guidelines addressed image use. I've learned from experience that it is far too easy to end up with a massive document file if the images used were too large for the document. I've also found the easiest way to check for this—or other things taking up too much space in a file—is to look at the file size.

The problem I ran into is that I can eyeball a file size and know if it's "not right", but I need to give a size number for others who don't have as much experience. Thankfully, I have a reasonable sample set: the source files for our large (over 400 pages) clinical protocol set. These documents are reasonably similar to most policy and procedure documents; some are single page, others are more involved, and while mostly text, there are images throughout.

The primary challenge was getting the dataset into a crunchable format. The rest is relatively easy number crunching.

## Approach One
My first approach, which did work, was cumbersome and complex. I wouldn't recommend it, but wanted to share it anyway in case pieces of it might have value.

I started with a shell command (in Zsh, on macOS): `ls -lR | pbcopy`. `ls` lists directory contents; the -l flag shows all of the file info, and the -R flag recursively tracks through all the subdirectories. I piped this into the `pbcopy` command which placed on the clipboard, and then pasted the output into Sublime Text (my text editor of choice).

The result looked something like this:

``` bash
total 29280
drwxr-xr-x   7 samuelkordik  staff       224 Apr 28  2021 1 - Introduction
drwxr-xr-x  11 samuelkordik  staff       352 Apr 29  2021 10 - Appendices
drwxr-xr-x  19 samuelkordik  staff       608 Apr 26  2021 2 - Clinical Policies
drwxr-xr-x  18 samuelkordik  staff       576 Dec 14  2020 3 - Operational Policies
drwxr-xr-x   9 samuelkordik  staff       288 Dec 13  2020 4 - Transport Destination Determination
...

./1 - Introduction:
total 872
-rwxr-xr-x@ 1 samuelkordik  staff  131297 Jun 18  2020 1 - Introduction.docx
-rwxr-xr-x  1 samuelkordik  staff   31051 Jun 18  2020 2 - Signature Page.docx
-rwxr-xr-x@ 1 samuelkordik  staff   37465 Jun 18  2020 3 - Delegation.docx
-rwxr-xr-x  1 samuelkordik  staff  189106 Jun 18  2020 3 - Delegation.pdf
-rwxr-xr-x  1 samuelkordik  staff   44044 Jun 18  2020 4 - Definitions.docx
...
```

One of my favorite tools in Sublime Text is the powerful regex find-and-replace functionality. Regex (short for "regular expression") is a concise way to define a pattern to look for in text. Think of it as wildcards on steroids. I was really only interested in the Word files; the fastest way to get just those was to "find all" (using a regex pattern), copy, then paste into a new document. This simple pattern did the trick: `^.*\.docx$`. In this, the "^" and "$" mark the beginning and end of a line. The `.` matches any character except a new line, and the `*` means an unlimited number of them (up until the .docx extension).

Once in the new file, I used a different pattern to find and replace: `^.*staff\s*(\d*).*\d{2}\s+\d{4} (.*\.docx)`. In regex, putting items in parentheses identifies specific groups, which can then be referenced in the replace criteria. The first group (`(\d*)`) grabs the size data; the second group (`(.*\.docx)`) gets the filename. In this regex, the first section (`^.*staff\s*`) matches everything up through the "staff" and the following the spaces. Then the size group, then `.*\d{2}\s+\d{4} ` gets the month and matches specifically for a two digit number (the day) and a four digit number (the year) separated by a space. Then the filename.

My replacement was `$2\t$1`, which puts the second group (the filename) first, then a tab, then the first group (size). The tab puts this into two columns in Excel when I paste it in.

Once in Excel, I could use standard stats formulas to look at the distribution. That's a complex process with a lot of moving parts, plus, Excel didn't really give me what I was looking for in a timely manner. Enter Approach Two

## Approach Two
After some time, I realized a better way to do this would be using the computing power of R paired with the useful fs package. For this, I created a shell script with a set of piped R functions that resulted in a far more useful data table. In additiona wide range of descriptive stat values, this approach also used the `fs::by` function to get human-readable size definitions.

Here's the shell script, with comments explaining each line.:
``` r
#!/usr/bin/env Rscript --vanilla

fs::dir_info(here::here(), type = "file", recurse = TRUE) |> # get file info. Same info as `ls -lR`, but in a dataframe.
    dplyr::mutate(ext = fs::path_ext(path)) |> # add a column with just the file extension
    dplyr::group_by(ext) |> # group by this column to get a summary row for each extension type.
    dplyr::summarize(n = dplyr::n(),                     # raw count of files
                     min = min(size),                    # minimum size
                     q25 = quantile(size, probs = .25),  # 25th percentile
                     med = median(size),                 # median (50th percentile)
                     q75 = quantile(size, probs = .75),  # 75th percentile
                     max = max(size),                    # maximum
                     mean = fs::as_fs_bytes(mean(size)), # mean (average)
                     sd = fs::as_fs_bytes(sd(size))) |>  # standard deviation
    dplyr::mutate(sd_1 = mean + sd, sd_2 = mean + sd*2, sd_3 = mean + sd*3) |> # add two and three standard deviations above mean
    dplyr::arrange(desc(n)) |> # sort in descending order of the count.
    print.data.frame(sigfig = 2)
```

The result of this was a very readable table full of useful statistics:
```r
    ext   n     min     q25     med     q75     max    mean      sd    sd_1
1  docx 577  18.08K  34.24K  35.98K  38.01K  35.52M 361.99K   2.31M   2.66M
2   pdf 297  50.85K 112.72K 137.51K 452.85K 128.71M   1.63M  10.68M  12.31M
3   jpg  24  52.34K 291.33K 384.71K 471.34K  531.1K 365.52K 124.26K 489.78K
4  dotx  10  28.51K  36.26K  72.41K  86.28K  86.37K  63.49K  26.41K  89.89K
5  pptx   6 517.79K   1.32M   5.66M  14.23M  28.44M   9.54M  11.05M  20.59M
6   zip   5  35.13M  38.21M  76.48M  83.03M 178.49M  82.27M     58M 140.27M
7  xlsx   4  22.33K  22.44K  22.59K  42.56K 102.14K  42.41K  39.82K  82.23K
8   wmv   3  81.73M  85.06M   88.4M 138.02M 187.63M 119.25M  59.31M 178.56M
9   png   2 205.95K 205.95K 205.95K 205.95K 205.95K 205.95K       0 205.95K
10  pub   2  150.5K 150.62K 150.75K 150.88K    151K 150.75K  362.04  151.1K
11  xls   2     36K     73K    110K    147K    184K    110K 104.65K 214.65K
12  mp4   1   9.17M   9.17M   9.17M   9.17M   9.17M   9.17M      NA      NA
13  svg   1 107.99K 107.99K 107.99K 107.99K 107.99K 107.99K      NA      NA
      sd_2    sd_3
1    4.98M   7.29M
2      23M  33.68M
3  614.04K  738.3K
4   116.3K 142.71K
5   31.64M  42.69M
6  198.26M 256.26M
7  122.05K 161.87K
8  237.87M 297.18M
9  205.95K 205.95K
10 151.46K 151.81K
11  319.3K 423.96K
12      NA      NA
13      NA      NA
```


