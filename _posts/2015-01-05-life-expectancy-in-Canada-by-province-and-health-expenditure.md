---
layout: post
title: Google Charts bindings for Shiny example - Life expectancy data visualization
description: Life expectancy in Canadian provinces by health expenditure data visualization
thumbnailurl: /images/bubblewaltz_th.png
category: articles
tags: [R, data-science, datavis, shiny]
published: true
---

Canadian Government [Open Data portal](http://data.gc.ca/eng) is the source of huge volumne of 
publicly available datasets collected by various federal and provincial departments and agencies.

In this post, I show how to acquire several datasets, clean up the data, and combine them into a
single, clean dataset ready to be used in effective visualization which I romantically named "bubble waltz".

The clean dataset consists of two categorical variables (Year and Province), two continuous variables (Health Expenditure and Population),
and Life Expectancy as an outcome. Motion chart is an ideal candidate for data visualization. I came across
an interesting [Google Charts bindings for Shiny](https://github.com/jcheng5/googleCharts) by Joe Cheng ([@jcheng5](https://twitter.com/jcheng)).

The result is "Life expectancy in Canadian Provinces by Health Expenditure" motion chart visualization 
deployed on [Shiny server](http://shiny.rstudio.com/). Just click the play button at the right bottom of app window to start the visualization.

<iframe src="http://opendata.rubyind.com:3838/opendata/" height="597" width="100%"></iframe>

Shiny app code is available on [GitHub](https://github.com/sasha-ruby/healthexp).

The app has also been listed in [Goverment of Canada Open Data portal Apps Gallery](http://open.canada.ca/en/apps/life-expectancy-canadian-provinces-health-expenditure).

## Data source

The data is obtained from the following data tables to combine life expectancy, 
health expenditure and population by Province into a single data set for visualization:

- [Table 051-0001](http://data.gc.ca/data/en/dataset/b6b9c9e7-bd58-4b2f-8a7a-97bf370a4880) - Estimates of population, 
by age group and sex for July 1, Canada, provinces and territories annual (persons)
- [Table 102-0511](http://data.gc.ca/data/en/dataset/d33eca94-b40d-4d68-8027-842b56feb130) - Life expectancy, 
abridged life table, at birth and at age 65, by sex, Canada, provinces and territories, *Terminated* annual (years)
- [Table 384-0041](http://data.gc.ca/data/en/dataset/43d39e2b-63dc-499d-8e90-be3928374233) - Detailed household final consumption expenditure,
 provincial and territorial annual (dollars x 1,000,000)

I have broken down the app in two steps:

- [Getting and cleaning data](#getting-and-cleaning-data)
- [Visualization (Shiny app)](#shiny-app)

## Getting and cleaning data

To avoid the app to download and process the data on each page load, I have created a preprocess.R script
to do the following:

- Download the datasets and load them into R.
- Subset the data to remove rows and columns not relevant for the app.
- Impute missing values.
- Merge datasets.
- Save the clean dataset as `.RData` object in directory where shiny app lives.

This way, the shiny app can quickly load clean dataset.

Here's the code (also available as [Gist](https://gist.github.com/sasha-ruby/1d835f7ac26161fa3257)).

{% highlight R linenos %}
setwd("~/Documents/work/projects/opendata/")
library(doBy)
library(data.table)
library(zoo)

# Download life expectancy dataset
temp <- tempfile()
download.file("http://www20.statcan.gc.ca/tables-tableaux/cansim/csv/01020511-eng.zip", temp)
lifeexp <- read.csv(unz(temp, "01020511-eng.csv"), header = TRUE)
unlink(temp)

# convert to data.table
lifeexp <- data.table(lifeexp)

# Subset rows
lifeexp <- lifeexp[GEO != "Canada"][AGE == "At birth"][SEX == "Both sexes"][UNIT == "Life expectancy"][Value != "x",]

# subset columns
lifeexp <- lifeexp[, AGE := NULL]
lifeexp <- lifeexp[, SEX := NULL]
lifeexp <- lifeexp[, UNIT := NULL]
lifeexp <- lifeexp[, Vector := NULL]
lifeexp <- lifeexp[, Coordinate := NULL]

lifeexp$Value <- as.numeric(as.character(lifeexp$Value))

# Cleanup "Northwest Territories" and "Nunavut"
lifeexp <- lifeexp[, Ref_Date_GEO := paste(Ref_Date, GEO)]
lifeexp <- lifeexp[Ref_Date_GEO != "1999 Nunavut"][Ref_Date_GEO != "1999 Northwest Territories"]
lifeexp <- lifeexp[, Ref_Date_GEO := NULL]
# subset separate records for "Northwest Territories" | GEO == "Nunavut", merge them to get an average
nt <- lifeexp[GEO == "Northwest Territories" | GEO == "Nunavut"]
nt <- nt[,Value := mean(Value), by="Ref_Date"]
nt <- nt[GEO == "Nunavut"]
lifeexp <- lifeexp[GEO != "Northwest Territories" & GEO != "Nunavut"]
lifeexp <- rbind(lifeexp, nt)
lifeexp[GEO == "Territories" | GEO == "Nunavut", GEO := "Northwest Territories including Nunavut"]
rm(nt)

# Impute missing Yukon values with average value of existing records
yt <- lifeexp[GEO == 'Yukon']
mn <- yt[, mean(yt$Value)]
yt <- rbind(yt, as.list(c(2005, "Yukon", mn)))
yt <- rbind(yt, as.list(c(2006, "Yukon", mn)))
yt$Value <- as.numeric(as.character(yt$Value))
lifeexp <- rbind(lifeexp, yt[Ref_Date == 2005 | Ref_Date == 2006,])
rm(yt)
rm(mn)

# Rename columns
setnames(lifeexp, "Ref_Date", "Year")
setnames(lifeexp, "GEO", "Province")
setnames(lifeexp, "Value", "Life.Expectancy")
lifeexp$Year <- as.factor(lifeexp$Year)


# Download health expenditure dataset
temp <- tempfile()
download.file("http://www20.statcan.gc.ca/tables-tableaux/cansim/csv/03840041-eng.zip",temp)
healthspend <- read.csv(unz(temp, "03840041-eng.csv"), header = TRUE)
unlink(temp)

# convert to data.table
healthspend <- data.table(healthspend)

# Subset rows
healthspend <- healthspend[GEO != "Canada"][GEO != "Outside Canada"][Ref_Date > 1990][Ref_Date < 2007]
healthspend <- healthspend[PRI == "Current prices"][EST == "Health (x 1,000,000)"][Value != ".."]

# subset columns and rename columns
healthspend <- healthspend[, PRI := NULL]
healthspend <- healthspend[, EST := NULL]
healthspend <- healthspend[, Vector := NULL]
healthspend <- healthspend[, Coordinate := NULL]
healthspend$Value <- as.numeric(as.character(healthspend$Value))
setnames(healthspend, "Ref_Date", "Year")
setnames(healthspend, "GEO", "Province")
setnames(healthspend, "Value", "Health.Expenditure")
healthspend$Year <- as.factor(healthspend$Year)

# Merge split values for NWT and NT for 1999 - 2006 period
nwt <- healthspend[Province == "Northwest Territories" | Province == "Nunavut"]
nwt$Health.Expenditure <- as.numeric(as.character(nwt$Health.Expenditure))
nwt <- nwt[, Health.Expenditure := sum(Health.Expenditure), by = c("Year")]
nwt <- nwt[Province == "Nunavut"]
nwt <- nwt[, Province := "Northwest Territories including Nunavut"]
healthspend <- healthspend[Province != "Northwest Territories" & Province != "Nunavut"]
healthspend <- rbind(healthspend, nwt)
rm(nwt)


# Download population dataset
temp <- tempfile()
download.file("http://www20.statcan.gc.ca/tables-tableaux/cansim/csv/00510001-eng.zip",temp)
population <- read.csv(unz(temp, "00510001-eng.csv"), header = TRUE)
unlink(temp)
rm(temp)

# convert to data.table
population <- data.table(population)

# Subset rows
population <- population[GEO != "Canada"][Ref_Date > 1990][Ref_Date < 2007][SEX == "Both sexes"][AGE == "All ages"]

# subset columns and rename columns
population <- population[, SEX := NULL]
population <- population[, AGE := NULL]
population <- population[, Vector := NULL]
population <- population[, Coordinate := NULL]

population$Value <- as.numeric(as.character(population$Value))
setnames(population, "Ref_Date", "Year")
setnames(population, "GEO", "Province")
setnames(population, "Value", "Population")
population$Year <- as.factor(population$Year)

nwt <- population[Province == "Northwest Territories" | Province == "Nunavut"]
nwt <- nwt[, Population := sum(Population), by = "Year"]
nwt <- nwt[Province == "Nunavut"]
nwt <- nwt[, Province := "Northwest Territories including Nunavut"]
population <- population[Province != "Northwest Territories" & Province != "Nunavut"]
population <- rbind(population, nwt)
rm(nwt)

# Merge datasets
dataset <- merge(lifeexp, healthspend, by=c("Year", "Province"))
dataset <- merge(dataset, population, by=c("Year", "Province"))

dataset$Year <- as.numeric(as.character(dataset$Year))
dataset$Country <- "Canada"
dataset$Country <- as.factor(dataset$Country)
setcolorder(dataset, c("Province", "Life.Expectancy", "Health.Expenditure", "Country", "Population", "Year"))

rm(lifeexp, healthspend, population)

dataset$ProvinceCode[dataset$Province == 'Alberta'] <- 'AB'
dataset$ProvinceCode[dataset$Province == 'British Columbia'] <- 'BC'
dataset$ProvinceCode[dataset$Province == 'Manitoba'] <- 'MB'
dataset$ProvinceCode[dataset$Province == 'New Brunswick'] <- 'NB'
dataset$ProvinceCode[dataset$Province == 'Newfoundland and Labrador'] <- 'NL'
dataset$ProvinceCode[dataset$Province == 'Northwest Territories including Nunavut'] <- 'NT & NU'
dataset$ProvinceCode[dataset$Province == 'Nova Scotia'] <- 'NS'
dataset$ProvinceCode[dataset$Province == 'Ontario'] <- 'ON'
dataset$ProvinceCode[dataset$Province == 'Prince Edward Island'] <- 'PEI'
dataset$ProvinceCode[dataset$Province == 'Quebec'] <- 'QC'
dataset$ProvinceCode[dataset$Province == 'Saskatchewan'] <- 'SK'
dataset$ProvinceCode[dataset$Province == 'Yukon'] <- 'YT'
dataset$Province <- as.factor(dataset$ProvinceCode)
dataset <- dataset[, ProvinceCode := NULL]

dataset$Health.Expenditure <- dataset$Health.Expenditure / dataset$Population * 1000000

save.image("./healthexp/healthexp.RData")
{% endhighlight %}


## Shiny app

Shiny app is split into:

*`global.R`* script to load required libraries and dataset.

{% highlight R linenos %}
suppressPackageStartupMessages(library(shiny))
suppressPackageStartupMessages(library(data.table))
suppressPackageStartupMessages(library(dplyr))
suppressPackageStartupMessages(library(googleVis))

# More info:
#   https://github.com/jcheng5/googleCharts
# Install:
#   devtools::install_github("jcheng5/googleCharts")
suppressPackageStartupMessages(library(googleCharts))

load("healthexp.RData", envir=.GlobalEnv)
{% endhighlight %}

*`ui.R`* script that binds the UI.

{% highlight R linenos %}
# Use global max/min for axes so the view window stays
# constant as the user moves between years
xlim <- list(
    min = 0,
    max = 1200    
)
ylim <- list(
    min = 72,
    max = 83
)

shinyUI(fluidPage(
    
    # This line loads the Google Charts JS library
    googleChartsInit(),
    
    # Use the Google webfont "Source Sans Pro"
    tags$link(
        href=paste0("http://fonts.googleapis.com/css?",
            "family=Source+Sans+Pro:300,600,300italic"),
        rel="stylesheet", type="text/css"
    ),
        
    tags$style(type="text/css",
        "body {font-family: Helvetica, Arial, 'Source Sans Pro'}"
    ),
    
    googleBubbleChart("chart",
        width="100%", height = "475px",
        # Set the default options for this chart; they can be
        # overridden in server.R on a per-update basis. See
        # https://developers.google.com/chart/interactive/docs/gallery/bubblechart
        # for option documentation.
        options = list(
            fontName = "sans-serif",
            fontSize = 13,
            # Set axis labels and ranges
            hAxis = list(
                title = "Health expenditure per capita (C$)",
                viewWindow = xlim
            ),
            vAxis = list(
                title = "Life expectancy (years)",
                viewWindow = ylim
            ),
            # The default padding is a little too spaced out
            chartArea = list(
                top = 50, left = 75,
                height = "75%", width = "75%"
            ),
            # Allow pan/zoom
            explorer = list(
                'keepInBounds' = TRUE,
                'maxZoomIn' = 1,
                'maxZoomOut' = 1
            ),
            # Set bubble visual props
            bubble = list(
                opacity = 0.60, 
                stroke = "none",
                textStyle = list(
                    fontSize = 11
                )
            ),
            # Set fonts
            titleTextStyle = list(
                fontSize = 16
            ),
            tooltip = list(
                textStyle = list(
                    fontSize = 12
                )
            ),
            colorAxis = list(
                colors = list(
                    'yellow', 'red', 'blue', 'green', 'orange', 'pink', 'grey', 'purple', 'magenta', 'cyan', 'beige', 'lavender', 'gold'
                )
            ),
            sizeAxis = list(
                minSize = 5,
                maxSize = 20,
                minValue = min(dataset$Population),
                maxValue = max(dataset$Population)
            ),
            animation = list(
                'duration' = 1000,
                'easing' = 'inAndOut'
            )
        )
    ),
    
    fluidRow(
        shiny::column(4, offset = 4,
            sliderInput("year", "Year",
                min = min(dataset$Year), 
                max = max(dataset$Year),
                value = min(dataset$Year), 
                animate = TRUE
            )
        )
    )
)){% endhighlight %}

*`server.R`* script to bind the data to UI elements.

{% highlight R linenos %}
shinyServer(function(input, output, session) {
    
    # Provide explicit colors for regions, so they don't get recoded when the
    # different series happen to be ordered differently from year to year.
    # http://andrewgelman.com/2014/09/11/mysterious-shiny-things/
    defaultColors <- c("#3366cc", "#dc3912", "#ff9900", "#109618", "#990099", "#0099c6", "#dd4477", 
                       "#3366cc", "#dc3912", "#ff9900", "#109618", "#990099", "#0099c6")
    series <- structure(
        lapply(defaultColors, function(color) { list(color=color) }),
        names = levels(dataset$Province)
    )
    
    yearData <- reactive({
        # Filter to the desired year, and put the columns
        # in the order that Google's Bubble Chart expects
        # them (name, x, y, color, size). Also sort by region
        # so that Google Charts orders and colors the regions
        # consistently.
        df <- dataset %.%
            filter(Year == input$year) %.%
            select(Province, Health.Expenditure, Life.Expectancy, NULL, Population)
    })
    
    output$chart <- reactive({
        # Return the data and options
        list(
            data = googleDataTable(yearData()),
            options = list(
                title = sprintf(
                    "Life Expectancy vs. Health Expenditure per Capita by Province, %s",
                    input$year),
                series = series
            )
        )
    })
    
})

{% endhighlight %}
