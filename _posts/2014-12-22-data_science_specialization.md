---
layout: post
title: Coursera Data Science Specialization
---

Wordiction is a text prediction app that predicts the next word to be typed in, based on previous text. It is developed in the scope of the Capstone Project for Coursera / John Hopkins Data Science Specialization.

It suggests three words most likely to be the next in the given context.

Prediction is based on corpus provided in Coursera Data Science Specialization Capstone project, which consists of texts extracted from extracts from Twitter, news and blog websites.

![Wordiction screenshot]({{ site.baseurl }}/images/wordiction.png)

### Preprocessing data

The following steps have been performed in data preprocessing step: 

- Load and concatenate twitter, blogs and news text files provided.
- Split the text into 10000 chunks for faster processing.
- Tokenize and create ngrams (bi-, tri- and four-gram) table with phrases and their frequences.
- Prune the ngrams table - strip ngrams with frequency less than 2.
- Strip all non-alphabetic characters (preserve line endings).
- Save ngrams tables into csv file.

### Building the model

The following algorithm is used in building the prediction model: 

- Read in the csv file with ngrams table and split into separate bi-, tri- and fourgrams data tables.
- Use back-off algorithm in the lookup process to suggest top 3 words that complete most common ngrams:
- Look up fourgrams that begin with the last 3 words typed-in.
- If not found in fourgrams table, lookup the trigrams that begin with the last 2 words.
- If not found in trigrams table, lookup the bigrams that begin with the last word.
- If not found in bigrams table, suggest the three most common unigrams.

### Wordiction app usage

Start typing in the Text box. The app provides three suggestions most likely to be the next word to be typed in.

The first suggestion is the most likely to be the one. To accept a suggestion, simply click the accpet button next to the best suggestion, and word is appended to the text area.

Click Clear button to clear the text from text box.