# Pittsburgh Air Quality Data Visualizations

This project was completed as part of CMU's Fall 2025 Data Visualizations course (05-619). The goal of this assignment was to gain familiarity implementing interaction techniques for visualizations using **Svelte** and **D3** and think critically about the effectiveness of specific techniques for a chosen data domain. We faced the additional challenge of scoping our assignment to provide a focused, welll-implemented interactive application within the deadline. This project was built using a starter file provided by Professor Moritz. 

[View Application →](https://kimcredit.github.io/Interactive-Visualization-Air-Quality/)




## First Visualization:

For the first visualization, we were tasked with creating a time-series visualization of air quality data using datasets from 8 locations around the Pittsburgh area. I began by reviewing the D3 and Svelte documentation, as well as relevant sources from lectures and labs, and online examples of visualizations with similar elements.
This visualization took me approximately 40 hours to complete, including initial work completed in a separate repository before the starter file was provided.

## Second Visualization:

#### Goal:

For the second visualization, my goal was to enable viewers to answer the question: **“What trends are there in AQI quality across seasons?”** I wanted viewers to be able to understand if there were any times of year they should be more cautious about the effects of air quality, especially for groups with increased sensitivity. I also wanted them to be able to compare the seasonal trends across the different station locations to see if the fluctuations differed or held steady. 

#### Development Process

I decided that I was going to display both visualizations on a single page to provide the most information to the viewer. This simplified some design decisions, since the necessity for visual consistency meant I needed to use shared visual encodings between both products. 

I also decided to find the mean AQI values for each day across all years in the dataset instead of keeping the years separate. This condensed, 1 year domain makes it easier to identify seasonal or annual trends, and only removes information that is already available in the first visualization. My x-axis has labels for months, which are also sorted into seasonal groupings. The months start with december and with november so the winter months are grouped together. 

For interactive elements, I used the same dropdown menu from the first visualization to select stations for the second visualization. I added a simple data filter via radio buttons that lets the viewer switch between viewing the mean AQI and max AQI for each day, to reveal any trends in AQI level spikes. I also added tooltips to provide the actual AQI numerical values to the viewer. The tooltips show the date(month and day), the mean AQI value, the max AQI value and its year, and the min AQI value and its year. 

I used shared widths, layout, and styling for both visualizations to make them appear more like one cohesive deliverable than two separate entities. 

I spent a little under 20 hours on this visualization, which was less time than I spent on the first since I became more familiar with the tools as I went. Across both visuals, the most time-consuming was the initial manipulation of the dataset, since digesting and expanding on the examples provided in the documentation required a lot of baseline programming knowledge that I am still new to. I spent more time overall producing the visuals and adjusting style elements, but it felt shorter because the process could be broken down into more manageable milestones.

#### Alternatives & Ultimate Choices: 

Originally I wanted viewers to be able to show the daily mean, max, and min AQI for each day using the data filter radio buttons. However, once I implemented this and tried it out, I realized the minimum AQI filter provided no additional value to the viewer because almost all of the values fell within the lowest color-range. Instead, I only provide the min AQI in the tooltip where viewers see exact value. 
I also altered my tooltips when I realized that, because four of the stations only have about a year’s worth of data, many of the days were showing the same values for the mean, max, and min AQI. I added a conditional that only shows the max and min values when there are multiple AQI values for that day. 

#### Sources and AI: 

In addition to documentation examples found online, and class resources, I also had help from my classmate Anissa Patel, who helped explain some of the elements of the starter code. She also helped guide me through the first few steps in the process by first letting me try myself, and then helping me identify and fix what I did wrong.

I used AI to provide explanations for lines of code I didn’t understand in the starter file or from examples online. I also used it to find sources to explain elements or concepts, and to debug my code when it wasn’t working. 

[See a list of my sources and AI prompts →](https://docs.google.com/document/d/19wHew1xW1okVK6ed1Fv7DuDDNpmB1yiIAi-h0D_fbi4/edit?usp=sharing) 
