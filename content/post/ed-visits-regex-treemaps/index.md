---
title: Regular Expression & Treemaps to Visualize Emergency Department Visits
author: Eugene
date: '2017-05-25'
slug: ed-visits-regex-treemaps
categories:
  - R
  - Public Health
tags: []
subtitle: ''
summary: ''
featured: false
image:
  caption: ''
  focal_point: ''
  preview_only: yes
projects: []
---

Original post date: May 25 2017

I had the opportunity to attend the [Open Data Science Conference](https://www.odsc.com/boston) (ODSC) East held in Boston, MA. Over a two day period I had the opportunity to listen to a number of leaders in various industries and fields. It was inspiring to learn about the wide variety of data science applications ranging from finance and marketing to genomics and even the refugee crisis.

One of the workshops at ODSC was text analytics, which includes basic text processing, dendrograms, natural language processing and sentiment analysis. This gave me the thought of applying some text analytics to visualize some data I was working on last summer. In this post I’m going to walk through how I used regular expression to label classification codes in a large dataset ([NHAMCS](https://www.cdc.gov/nchs/ahcd/about_ahcd.htm)) representing emergency department visits in the United States and eventually visualize the data.

## The NHAMCS and ICD-9 Codes
Without going into too much detail, the **National Health Ambulatory Medical Care Survey** (NHAMCS) is large sample survey used to provide an estimate of visits to outpatient and emergency departments in general/short-stay hospital at a national scale for the United States. Those who are well versed in complex probability-sample survey designs (or interested in it), you can read about how it’s set up [here](https://www.cdc.gov/nchs/ahcd/ahcd_scope.htm). For this post I only using the emergency department data, not outpatient. The main thing to keep in mind moving forward is that the survey’s basic unit of measurement are visits or encounters made in the United States to emergency departments.

Diagnostic information on each visit is collected and coded in the NHAMCS by using the [International Classification of Diseases](https://en.wikipedia.org/wiki/International_Statistical_Classification_of_Diseases_and_Related_Health_Problems) (ICD) codes maintained by the World Health Organization. Specifically the NHAMCS public data files (1992-2014) use the [ICD, 9th revision](https://www.cdc.gov/nchs/icd/icd9cm.htm), clinical modification (ICD-9-CM). The 10th revision being currently used throughout the world since 2015 and the 11th revision is in-process, most likely it will be disseminated in 2018. You can check out this [package](https://jackwasey.github.io/icd/) I recently came across that validates and searches co-morbidities of ICD-9 and ICD-10 codes in R.

## The Data Game Plan
In this post, the code is only for the 2005 emergency department (ED) NHAMCS dataset. If you interested in getting the original data you can go to [this page](https://www.cdc.gov/nchs/ahcd/ahcd_questionnaires.htm#public_use). There is extensive documentation on the NHAMCS website that helps you download the data for use in SAS, Stata and SPSS. From my current knowledge (hours searching the vast interweb) there is no method in R to download these files. If someone wants to take the challenge of developing a method of pulling down the data using R, go for it! For this post, the data I’m using were Stata files and I used an R package to read-in those files. I’ve included subsetted data for ED NHAMCS from 2005 to 2009 in my [GitHub repository](https://github.com/eugejoh/ICD-9_DV) if anyone is interested in using those files with my code.

To achieve our goal of visualizing diagnoses in ED visits we need to obtain, clean, and then appropriately setup our data. First we’re going to parse the text in official ICD-9-CM documentation using regular expression to create our own ICD dictionary/lookup table. Next, we’ll take the 2005 ED NHAMCS data, apply the survey design to get the correct estimates per ED visit. Then we’ll use our ICD dictionary to define each code, set it all up using the `data.table` [package](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-intro.html) to create a [Treemap](https://en.wikipedia.org/wiki/Treemapping) visualization. **I’ll be explaining a fair amount on data cleaning and regular expression, if you want to see the treemap you can scroll down to the end of the post.**

## Using Regex on ICD-9 Codes

Regular expression or regex is not unique just to R but is used across other types of programming languages (Java, Python, Perl). Simply put, regex is used to extract patterns of characters in a string. There is a good tutorial video [here](https://www.youtube.com/watch?v=q8SzNKib5-4) on regex and another video [here](https://www.youtube.com/watch?v=NvHjYOilOf8) on using regex in R. Disclaimer: I am not an expert on regex! I am still learning and this post is based on my current knowledge I’ve gathered so far. A good and helpful source I’ve been using  is [Rexegg](http://www.rexegg.com/). Regardless, let’s march forward towards our goal!

I’m going to take the 3-digit grouping ICD-9-CM documentation from the CDC’s Health Statistics [website]ftp://ftp.cdc.gov/pub/Health_Statistics/NCHS/Publications/ICD9-CM/2010), where I downloaded the `DINDEX11.zip` file. Because of the annoying nature of the rich-text files (RTF), I just copied the entire document into a basic UTF-8 text file called `Dc_3d10.txt`. My entire code is in my GitHub, but I’ll be referring to particular sections in this post. So moving forward, I’ll read this into R using the `readLines` function and this output...

```r
icd9.3 <- readLines("Dc_3d10.txt") > head(icd9.3,15)
#  [1] "Appendix E\u2028List of Three-Digit Categories"
#  [2] "1.\tINFECTIOUS AND PARASITIC DISEASES"
#  [3] "Intestinal infectious diseases (001-009)"
#  [4] "001\tCholera"
#  [5] "002\tTyphoid and paratyphoid fevers"
#  [6] "003\tOther salmonella infections"
#  [7] "004\tShigellosis"
#  [8] "005\tOther food poisoning (bacterial)"
#  [9] "006\tAmebiasis"
# [10] "007\tOther protozoal intestinal diseases"
# [11] "008\tIntestinal infections due to other organisms"
# [12] "009\tIll-defined intestinal infections"
# [13] "Tuberculosis (010-018)"
# [14] "010\tPrimary tuberculous infection"
# [15] "011\tPulmonary tuberculosis"
```

It looks like there is a header/title at [1], numeric grouping  at [2] "1.\tINFECTIOUS AND PARASITIC DISEASES",  subgrouping by ICD-9 code ranges, at [3] "Intestinal infectious diseases (001-009)" and then 3-digit ICD-9 codes followed by a specific diagnosis, at [10] "007\tOther protozoal intestinal diseases". At the end we want to produce three separate data frames that we’ll categorize as:

 1. Groups: the title which contains the general diagnosis grouping  
 2. Subgroups: the range of ICD-9 codes that contain a certain diagnosis subgroup  
 3. Classification: the specific 3-digit ICD-9 code that corresponds with a diagnosis  
 
First we’ll use a simple operation to remove the first line:

```r
icd9.3 <- icd9.3[-1]
```

Next we’ll use regex to extract the Groups using this…

```r
icd9.3[grep("^[0-9]+\\.",icd9.3)]
```

The `grep()` function is used to extract patterns based on regex. To explain the regex, I am selecting lines that start with a digit of any length which is immediately followed by a period. The carat `^` initiates the pattern matching from the start of the string (going left to right). The `[0-9]` represents any digit from 0 through 9 and the plus sign `+` denotes at least one repeat of any digit. The two backslashes `\\` is required because periods in regex are special meta-characters that represent any character (number, letter, symbol). To summarize, this identifies strings that start with `1.`, `2.`, `3.`, ..., `17.`, and beyond.

Next I use the `gsub()` function, which replaces or substitutes based on regex to replace all the `\t` in the group names and replace it with two underscore `__` which we’ll use later as an identifier to split the string in half to make the data frame (if this isn’t clear right now, hopefully it will be when we do the splitting).


```r
icd9.3g <- gsub("\\.\t","__",icd9.3g) > icd9.3g[1:3]
# [1] "1__INFECTIOUS AND PARASITIC DISEASES"
# [2] "2__NEOPLASMS"
# [3] "3__ENDOCRINE, NUTRITIONAL AND METABOLIC DISEASES, AND IMMUNITY DISORDERS"
```
This regex removes all `\t` specifically when it follows a period. Here we use the `\\` again to identify a period.

Now we’ll split the string into separate columns within a data frame using the `str_split_fixed` function in the `stringr` [package](https://cran.r-project.org/web/packages/stringr/vignettes/stringr.html). I set `stringsAsFactors = FALSE` so that we maintain the data type of character with the strings. You can read about how this code has caused headaches for many users [here](https://www.r-bloggers.com/one-solution-to-the-stringsasfactors-problem-or-hell-yeah-there-is-hellno/). Here I assign `Group.n` for the number


```r
diag.g <- as.data.frame(stringr::str_split_fixed(icd9.3g,"__",2),stringsAsFactors = F)
names(diag.g) <- c("Group.n","Group")
diag.g <- diag.g[c("Group","Group.n")] #swap order of columns > head(diag.g)
#                                                                   Group Group.n
# 1                                     INFECTIOUS AND PARASITIC DISEASES     1
# 2                                                             NEOPLASMS     2
# 3 ENDOCRINE, NUTRITIONAL AND METABOLIC DISEASES, AND IMMUNITY DISORDERS     3
# 4                            DISEASES OF BLOOD AND BLOOD-FORMING ORGANS     4
# 5                                                      MENTAL DISORDERS     5
# 6                       DISEASES OF THE NERVOUS SYSTEM AND SENSE ORGANS     6
```

Now we have a two-column data frame that contains the group number and name of that group so we can refer to for the future. However, later on (I realized when I got there) I needed to create additional columns to account for the range of 3-digit values that correspond to each Group. To do this I used the `grep()` function to first get the names of each Group and the index of the Group titles in the text file by changing the value argument in `grep()`.

```r
groupn <- grep("\\.\t",icd9.3,value=T) #names
groupi <- grep("\\.\t",icd9.3,value=F) #index
```

Afterwards I find the lower and upper limit of each Group range by using the index and simple vector arithmetic in combination of `gsub()` to extract only numbers. For the upper limit and added the maximum value of 999 (only 3-digits remember!) so that the lengths match, since I didn’t extract it from the code above. Then create the final dictionary for the Groups.

```r
low.g <- gsub("[^0-9]","",icd9.3[groupi+2]) #lower limit of range
 
up.g <- c(gsub("[^0-9]","",icd9.3[groupi-1]),"999") #upper limit of range
 
diag.g <-data.frame(diag.g,low.g,up.g, stringsAsFactors = F)
names(diag.g) <- c("Group","Group.n","Start","End")
```

We move on to the Subgroups which we will extract based on the fact that all the corresponding lines have a parentheses contain a pattern of... 3 digits, a dash then another 3 digits. Ie. *"Fracture of skull (800-804)"* The appropriate regex is as follows...


```r
icd9.3sg <- icd9.3[grep("\\([0-9]{3}-[0-9]{3}\\)",icd9.3)]
```

The double backslashes are used for parentheses (just like for periods) and I specify the pattern of any 3 digits using the `{}` curly brackets. Next we remove any lingering `\t` that might exist and replace with a space `" "`. After that we’ll insert the `__` split identifier we used above. This is specified by using regex to match the pattern of 3-digit parentheses that follows directly after a space, which is denoted as `\\s`.


```r
icd9.3sg <- gsub("\t"," ",icd9.3sg)
icd9.3sg <- gsub("\\s\\(([0-9]{3}-[0-9]{3})\\)","__\\1",icd9.3sg)
```
Like we did for the Group above, we’ll split the based on the `__` split identifier.

```r
diag.sg <- as.data.frame(
  stringr::str_split_fixed(icd9.3sg,"__",2),
  stringsAsFactors = F)
```

After that, we even want to split the ranges by the dash `-` so we can get the ‘start’ and ‘end’ of the ICD-9 code range for each Subgroup. Then we’ll assign the names of the columns in the newly created data frame.

```r
diag.sg <- as.data.frame(
  cbind(
    diag.sg,
    stringr::str_split_fixed(diag.sg[,2],"-",2),
  stringsAsFactors = F))
names(diag.sg) <- c("Subgroup","Range","Start","End") > head(diag.sg,4)
                        # Subgroup &nbsp; Range Start End
# 1 Intestinal infectious diseases 001-009   001 009
# 2                   Tuberculosis 010-018   010 018
# 3    Zoonotic bacterial diseases 020-027   020 027
# 4       Other bacterial diseases 030-041   030 041
```

Now we’ll extract the Classifications. We want to generate a regex that captures a pattern of:

 1. A 3-digit sequence followed by a `\t`   
 
We can use the “|” or the OR operator in regex to capture these 3 patterns. Using similar regex from the Groups and Subgroups we generate this...

```r
icd9.3c <- icd9.3[grep("(^[0-9]{3}\t)|(^V[0-9]{2}\t)|(E[0-9]{3}\t)",icd9.3)]
```

And then we can do the same method as the Group to remove any lingering “\t” and split the string to get an output of a data frame. Also


```r
icd9.3c <- gsub("\t","__",icd9.3c)
diag.c <- as.data.frame(stringr::str_split_fixed(icd9.3c,"__",2),stringsAsFactors = F)
names(diag.c) <- c("Code","Classification")
diag.c <- diag.c[c("Classification","Code")] > head(diag.c)
#                     Classification Code
# 1                          Cholera  001
# 2   Typhoid and paratyphoid fevers  002
# 3      Other salmonella infections  003
# 4                      Shigellosis  004
# 5 Other food poisoning (bacterial)  005
# 6                        Amebiasis  006
```

We started with some messy text but now after all that regex we have successfully extracted 3 separate data frames for the ICD-9 Group, Subgroup and Classification that can used as a reference moving forward!

## Applying Survey Design
Since this post is focusing on regex, I will not go into detail about applying survey design to NHAMCS data files using R. In short, I used the `survey` [package](http://r-survey.r-forge.r-project.org/survey/) to apply the appropriate weighting based on the [4-stage probability sample survey](https://www.cdc.gov/nchs/ahcd/ahcd_scope.htm#nhamcs_scope) implemented by the NHAMCS. I’ve annotated my code for those who are interested about it but for the sake of this post I will not talk about it any further. All the details are based in the documentation files, that you can find by year in this [link](ftp://ftp.cdc.gov/pub/Health_Statistics/NCHS/Dataset_Documentation/NHAMCS).

Why do we have to apply the weighting? By doing this, we can actually obtain accurate national estimates of ED visits in the United States. If we started running descriptive statistics on our data without this step, we’ll only be analyzing unweighted counts, which might not be an accurate representation of the sampling population. For those interested can read this post on about weighted surveys in R [here](https://www.r-bloggers.com/social-science-goes-r-weighted-survey-data/).

## Matching Codes to Descriptions

For the sake of this analysis we’ll only look at the first column of ICD-9 codes in the 2005 ED NHAMCS dataset, denoted as `DIAG1`.

```r
# > ed05$DIAG1[1:8]
# [1] "8472-" "78099" "V997-" "V997-" "7931-" "8489-" "920--" "8920-"
```

At first glance we notice two main things...

First, that each set of strings are five characters long. This is because the ICD-9 code system allows for more specificity with diagnoses beyond 3-digits. Take a quick look at the ICD-9 [Wiki page](https://en.wikipedia.org/wiki/List_of_ICD-9_codes_390%E2%80%93459:_diseases_of_the_circulatory_system#Hypertensive_disease_.28401.E2.80.93405.29) for hypertensive disease. You can see that secondary hypertension can be further classified to malignant vs. benign and again whether it is renovascular or not. These classifications use up 5 digits (if you ignore the periods).

Secondly, that there are dashes when there are less than five characters. If we were only working with uniform patterns the `match` function would have been a simple solution (link to a short explanatory [post](https://www.r-bloggers.com/match-function-in-r/)), but this is not the case. It looks like we’ll have to use regex again to look at the first 3 characters in each element of the `DIAG1` vector.

I created a loop to match the 3-digit Classification code to the `DIAG1` column. I created a new column and filled it with the appropriate code using regex matching. The last two lines are included to help us finalize our dictionary using `data.table` package.


```r
ed05.df$Code <- "-"
for (k in paste0("^",diag.c[,2])){ #codes
  gindex <- grep(k,ed05.df[,1]) #matches code with DIAG1 column
  ed05.df$Code[gindex] <- diag.c$Code[grep(k,diag.c[,2])]
  ed05.df$Code.n <- as.numeric(ed05.df$Code) #some values are coered to NA
  ed05.df$Code.n[which(is.na(ed05.df$Code.n))] <- 1000 #treat all NA's wih value 1000
}
```

Then we use the `join()` function from the `plyr` package to match the Classifications with the codes in our new data frame.

```r
ed05.df <- join(ed05.df,diag.c,by="Code")
```

## Overlap Joins using `data.table`
I’m not going to get into much detail here because this was something I had to learn when I discovered that joins and merges based on character types (ie. “001” or “057”) were tedious and very annoying. Thankfully this StackOverflow [post](https://stackoverflow.com/questions/24480031/roll-join-with-start-end-window) was extremely useful to understand how the `foverlaps()` [function](https://www.rdocumentation.org/packages/data.table/versions/1.10.4/topics/foverlaps) works. In short to address this issue, I created numeric equivalents of the ICD-9 codes (ie. “019” became 19) because foverlaps() only works with numeric and integer values. After that I assigned the appropriate categories of Group and Subgroup to the Classification codes, cleaned it up a bit to remove unnecessary columns, and convert it back into a `data.frame` class. My code is annotated if you would want to look at it and I would refer to the post I mentioned above.

## Visualizing the Data
Since we have somewhat hierarchical data we can visualize this using a treemap. Now we make sure the `treemap` [package](https://cran.r-project.org/web/packages/treemap/index.html) is loaded and then use the lovely [IWantHue](http://tools.medialab.sciences-po.fr/iwanthue/) website to get a nice color palette. I’ve annotated the code to explain what each argument does and after running this we get...


```r
tree.n <- treemap(df.ed05,
  index = c("Group", "Subgroup", "Classification"), #grouping for each
  vSize = "Freq", #area of rectangles by NHAMCS estimates
  type = "index", #see documentation
  title = "NHAMCS Emergency Department Visits in 2005",
  overlap.labels = 0.5, ##see documentation
  palette = pal1, #custom palette from IWantHue
  border.col = c("#101010", "#292929", "#333333"), #set border colors
  fontsize.title = 40, #set font size
  fontsize.labels = c(30, 24, 16), #set size for each grouping
  lowerbound.cex.labels = .4, ##see documentation
  fontcolor.labels = c("#000000", "#292929", "#333333"), #set color
  fontface.labels = c(2, 4, 1), # bold, bold-italic, normal
  fontfamily.labels = c("sans"), #sans font
  inflate.labels = F, #see documentation
  align.labels = c("center", "center"), #align all labels in center
  bg.labels = 230 # 0 and 255 that determines the transparency
)
```

![](/post/ed-visits-regex-treemaps/icd-9_tree_2005.png){width=95%}

From this we see that most ED visits had a diagnosis related to injury, poisoning, symptoms and respiratory system issues. This makes sense because usually you go to the hospital when you have an injury or some acute medical emergency. Below I’ve also created the treemaps for the other years leading up to 2009 with the colors corresponding with each group.

![](/post/ed-visits-regex-treemaps/icd-9_tree_2005.png){width=20%}
![](/post/ed-visits-regex-treemaps/icd-9_tree_2006.png){width=20%}
![](/post/ed-visits-regex-treemaps/icd-9_tree_2007.png){width=20%}
![](/post/ed-visits-regex-treemaps/icd-9_tree_2008.png){width=20%}
![](/post/ed-visits-regex-treemaps/icd-9_tree_2009.png){width=20%}

Some comments on the data and visualization:

 - It does not include ICD-9 codes for Supplementary Classifications (codes that start with V’s and E’s). I ignored these due to tedious nature of matching the Groups with characters and maybe in the future I can write something that includes them to the visualization.  
 - This data is for all visits regardless of demographic information. I suspect the treemaps between age, gender, race, and region would look very different.  
 - These are national estimates and the true value is contained within a confidence interval dependent on the survey design. Other visualizations may be appropriate to show the probability ranges of this data.  
 - There were missing or blank ICD-9 codes (the coders at the hospitals either didn’t fill them or there were no diagnoses?) accounted for 2.78% of all estimates.  
 - Originally I wanted to create a radial or sunburst tree of the data, but I had difficulty setting up the data in an appropriate form (ie. classes Node to phylo). I would appreciate any knowledge on how to get around this and maybe make that my next mini-project.  
 - This data is from ~10 years ago, it would be interesting to see what the most recent NHAMCS data says about emergency department visits.  
 
## Final Thoughts
From ODSC a speaker said that for data visualization, 80% of your work is cleaning setting up the data for the visualization and 20% is actually making the visualization… very true for this case. I did not expect the regex and data.table to take as long as it did. Regardless, I got the chance to learn more about the data.table package, practice my regular expression knowledge, and explore different tree visualizations! Any feedback, comments, and questions on the code or visualization are always welcome!


![](/post/ed-visits-regex-treemaps/odsc.jpg){width=20%}
