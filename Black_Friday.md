Black Friday: EDA, Spending Prediction and Product Recommender
================
[Hendrik Mischo](https://github.com/hendrik-mischo), [Kelly Raas](https://github.com/kellyraas)
22 January 2019

Introduction
============

The dataset at hand is comprised of roughly 558,000 observations about Black Friday shoppers in a retail store. You can find the dataset and more detailed information [here](https://www.kaggle.com/mehdidag/black-friday).

The objectives of this project are the following:
1. Discover the underlying patterns of the data and derive insights about customers and products, which are useful to the marketing team and management.
2. Create a model to predict how much a given customer will likely spend.
3. Build a [Recommendation system](https://en.wikipedia.org/wiki/Recommender_system) that recommends new products to customers, which they are likely to purchase.

To achieve the goal \#1 we will perform an Exploratory Data Analysis (EDA) using the Tidyverse library for data wrangling and visualizations.

Given that goal \#2 can be an extensive project by itself, we will only scratch the surface here. However, we will establish the baseline performance of a regression model and a tree-based model and provide suggestions on how the performance could be further improved.

Goal \#3 is going to be achieved by with the [Collaborative Filtering (CF)](https://en.wikipedia.org/wiki/Collaborative_filtering) technique. We will use the Recommenderlab library and compare Item Based Collaborative Filtering (IBCF) and User Based Collaborative Filtering (UBCF) to find the optimal method.

Data Cleaning
=============

Let's start by importing the relevant libraries and the data.

``` r
library(tidyverse)      # For data wrangling and visualization
library(gridExtra)      # To arrage plots
library(scales)         # For different scales e.g. US-dollars
library(recommenderlab) # For Collaborative Filtering
library(caret)          # For Linear Regression
library(randomForest)   # For Random Forests
library(ggthemes)       # For themes

# Import the data
data = read.csv("data/blackfriday.csv")
attach(data)

# Define the theme to be applied to the plots
theme.hc = function(){
  theme(
    text                = element_text(size = 12),
    title               = element_text(hjust = 0), 
    axis.title.x        = element_text(hjust = .5),
    axis.title.y        = element_text(hjust = .5),
    panel.grid.major.y  = element_line(color = 'gray', size = .15),
    panel.grid.minor.y  = element_blank(),
    panel.grid.major.x  = element_blank(),
    panel.grid.minor.x  = element_blank(),
    panel.border        = element_blank(),
    panel.background    = element_blank(),
    legend.position     = "right",
    legend.title        = element_blank())}
```

Now let's glimpse the data.

``` r
head(data)
```

      User_ID Product_ID Gender   Age Occupation City_Category
    1 1000001  P00069042      F  0-17         10             A
    2 1000001  P00248942      F  0-17         10             A
    3 1000001  P00087842      F  0-17         10             A
    4 1000001  P00085442      F  0-17         10             A
    5 1000002  P00285442      M   55+         16             C
    6 1000003  P00193542      M 26-35         15             A
      Stay_In_Current_City_Years Marital_Status Product_Category_1
    1                          2              0                  3
    2                          2              0                  1
    3                          2              0                 12
    4                          2              0                 12
    5                         4+              0                  8
    6                          3              0                  1
      Product_Category_2 Product_Category_3 Purchase
    1                 NA                 NA     8370
    2                  6                 14    15200
    3                 NA                 NA     1422
    4                 14                 NA     1057
    5                 NA                 NA     7969
    6                  2                 NA    15227

Check how the values are distributed and if there are any abnormal values.

``` r
summary(data)
```

        User_ID            Product_ID     Gender        Age        
     Min.   :1000001   P00265242:  1858   F:132197   0-17 : 14707  
     1st Qu.:1001495   P00110742:  1591   M:405380   18-25: 97634  
     Median :1003031   P00025442:  1586              26-35:214690  
     Mean   :1002992   P00112142:  1539              36-45:107499  
     3rd Qu.:1004417   P00057642:  1430              46-50: 44526  
     Max.   :1006040   P00184942:  1424              51-55: 37618  
                       (Other)  :528149              55+  : 20903  
       Occupation     City_Category Stay_In_Current_City_Years
     Min.   : 0.000   A:144638      0 : 72725                 
     1st Qu.: 2.000   B:226493      1 :189192                 
     Median : 7.000   C:166446      2 : 99459                 
     Mean   : 8.083                 3 : 93312                 
     3rd Qu.:14.000                 4+: 82889                 
     Max.   :20.000                                           
                                                              
     Marital_Status   Product_Category_1 Product_Category_2 Product_Category_3
     Min.   :0.0000   Min.   : 1.000     Min.   : 2.00      Min.   : 3.0      
     1st Qu.:0.0000   1st Qu.: 1.000     1st Qu.: 5.00      1st Qu.: 9.0      
     Median :0.0000   Median : 5.000     Median : 9.00      Median :14.0      
     Mean   :0.4088   Mean   : 5.296     Mean   : 9.84      Mean   :12.7      
     3rd Qu.:1.0000   3rd Qu.: 8.000     3rd Qu.:15.00      3rd Qu.:16.0      
     Max.   :1.0000   Max.   :18.000     Max.   :18.00      Max.   :18.0      
                                         NA's   :166986     NA's   :373299    
        Purchase    
     Min.   :  185  
     1st Qu.: 5866  
     Median : 8062  
     Mean   : 9334  
     3rd Qu.:12073  
     Max.   :23961  
                    

Check the variable types.

``` r
glimpse(data)
```

    Observations: 537,577
    Variables: 12
    $ User_ID                    <int> 1000001, 1000001, 1000001, 1000001,...
    $ Product_ID                 <fct> P00069042, P00248942, P00087842, P0...
    $ Gender                     <fct> F, F, F, F, M, M, M, M, M, M, M, M,...
    $ Age                        <fct> 0-17, 0-17, 0-17, 0-17, 55+, 26-35,...
    $ Occupation                 <int> 10, 10, 10, 10, 16, 15, 7, 7, 7, 20...
    $ City_Category              <fct> A, A, A, A, C, A, B, B, B, A, A, A,...
    $ Stay_In_Current_City_Years <fct> 2, 2, 2, 2, 4+, 3, 2, 2, 2, 1, 1, 1...
    $ Marital_Status             <int> 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1,...
    $ Product_Category_1         <int> 3, 1, 12, 12, 8, 1, 1, 1, 1, 8, 5, ...
    $ Product_Category_2         <int> NA, 6, NA, 14, NA, 2, 8, 15, 16, NA...
    $ Product_Category_3         <int> NA, 14, NA, NA, NA, NA, 17, NA, NA,...
    $ Purchase                   <int> 8370, 15200, 1422, 1057, 7969, 1522...

See if there are any NAs.

``` r
sapply(data, function(x) sum(is.na(x)))
```

                       User_ID                 Product_ID 
                             0                          0 
                        Gender                        Age 
                             0                          0 
                    Occupation              City_Category 
                             0                          0 
    Stay_In_Current_City_Years             Marital_Status 
                             0                          0 
            Product_Category_1         Product_Category_2 
                             0                     166986 
            Product_Category_3                   Purchase 
                        373299                          0 

We notice the following issues:
1) Some numeric variables should be factors.
2) `User_ID` and `Product_ID` should be characters.
3) There are quite a lot of NAs in `Product_Category_2` and `Product_Category_3`.
4) `Purchase` seems to be formatted in cents given the high values.
Next we will clean the data. The missing product categories will be represented as category 0. `Purchase` will be converted to dollars.

``` r
data = data %>%
  mutate(
    # Convert several variables to character
    User_ID = as.character(User_ID),
    Product_ID = as.character(Product_ID),
    # Convert several variables to factors
    Occupation = as.factor(Occupation),
    Marital_Status = as.factor(Marital_Status),
    Product_Category_1 = as.factor(Product_Category_1),
    Product_Category_2 = as.factor(Product_Category_2),
    Product_Category_3 = as.factor(Product_Category_3),
    # Missing product categories will be represented as category 0
    Product_Category_2 = ifelse(is.na(Product_Category_2), 0, Product_Category_2),
    Product_Category_3 = ifelse(is.na(Product_Category_3), 0, Product_Category_3),
    # Convert cents to dollars
    Purchase = as.numeric(Purchase)/100 
    )
attach(data)
```

Let's have a look at the clean data.

``` r
head(data)
```

      User_ID Product_ID Gender   Age Occupation City_Category
    1 1000001  P00069042      F  0-17         10             A
    2 1000001  P00248942      F  0-17         10             A
    3 1000001  P00087842      F  0-17         10             A
    4 1000001  P00085442      F  0-17         10             A
    5 1000002  P00285442      M   55+         16             C
    6 1000003  P00193542      M 26-35         15             A
      Stay_In_Current_City_Years Marital_Status Product_Category_1
    1                          2              0                  3
    2                          2              0                  1
    3                          2              0                 12
    4                          2              0                 12
    5                         4+              0                  8
    6                          3              0                  1
      Product_Category_2 Product_Category_3 Purchase
    1                  0                  0    83.70
    2                  5                 11   152.00
    3                  0                  0    14.22
    4                 13                  0    10.57
    5                  0                  0    79.69
    6                  1                  0   152.27

Exploratory Data Analysis
=========================

This section is divided into two parts. We will first analyze the customers and then the products.

Customer Analysis
-----------------

What can we find out about our customers?

### How many unique customers are there?

``` r
data %>% distinct(User_ID) %>% nrow()
```

    [1] 5891

### Gender Distribution

``` r
data %>%
  group_by(Gender) %>%
  summarise(total = n_distinct(User_ID)) %>%
  mutate(share = total/sum(total)) %>%
  ggplot(aes(x = "", y = total, fill = Gender)) +
  geom_bar(width = 1, stat = "identity", color = "white") +
  coord_polar("y", start=0) +
  geom_text(aes(label = paste0(round(share*100), "%")), position = position_stack(vjust = 0.5)) +
  labs(title = "Gender Distribution") +
  theme_void() +
    theme(plot.title = element_text(hjust = 0.5)) +
    theme(legend.position="right")
```

<img src="files/unnamed-chunk-9-1.png" style="display: block; margin: auto;" />

Men are significantly more respresented than women in this dataset.

### Age Distribution

``` r
data %>%
  group_by(Age) %>%
  summarize(n.customers = n_distinct(User_ID)) %>%
  ggplot(aes(x = Age, y = n.customers, fill = Age)) +
  geom_histogram(stat = "identity") +
  ggtitle("Age Distribution") +
  xlab("Age Group") +
  ylab("Number of Customers") +
  guides(fill = FALSE) +
  theme.hc()
```

<img src="files/unnamed-chunk-10-1.png" style="display: block; margin: auto;" />

The majority of customers is of age between 18 and 45.

### Average Spending by Age Group and Gender

``` r
data %>%
  group_by(Age, Gender) %>%
  summarise(purchase = sum(Purchase),
            count = n_distinct(User_ID),
            average = purchase / count) %>%
  ggplot(aes(x = Age, y = average, fill = Gender)) +
  scale_y_continuous(labels = dollar, breaks = seq(0, 10000, 1000)) +
  geom_bar(stat = "identity", position = "dodge") +
  ggtitle("Average Spending per Age Group and Gender") +
  xlab("Age") +
  ylab("Average Spending") +
  theme.hc()
```

<img src="files/unnamed-chunk-11-1.png" style="display: block; margin: auto;" />

It appears that on average men spend significantly more than women in all age groups and that people between 18 and 45 spend the most.

### Average Number of Transactions by Age Group and Gender

``` r
data %>%
  group_by(Age, Gender) %>%
  summarize(transactions = n(),
            count = n_distinct(User_ID),
            avg.transactions = transactions / count) %>%
  ggplot(aes(x = Age, y = avg.transactions, fill = Gender )) +
  geom_bar(stat = "identity", position = "dodge") +
  scale_y_continuous(breaks = seq(0, 120, 15))  +
  ggtitle("Average Number of Transactions per Age Group and Gender") +
  xlab("Age") +
  ylab("Average Number of Transactions") +
  theme.hc()
```

<img src="files/unnamed-chunk-12-1.png" style="display: block; margin: auto;" />

The average number of transactions is almost identically distributed as the average spending. This indicates that a higher spending does not mean that customers bought more expensive items, but just more items in general.

### Average Spending by Marital Status

``` r
data %>%
  group_by(Marital_Status) %>%
  summarize(purchase = sum(as.numeric(Purchase)),
            count = n_distinct(User_ID),
            average = purchase / count) %>%
  ggplot(aes(x = Marital_Status, y = average, fill = Marital_Status)) +
  geom_bar(stat = "identity") +
  scale_fill_discrete(labels = c("Unmarried", "Married")) +
  ggtitle("Average Spending per Marital Status") +
  xlab("Marital Status") +
  ylab("Average Spending") +
  theme.hc()
```

<img src="files/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

It appears that on average unmarried customers spend slightly more than married customers.

### Customer and Purchase Amount by City Category

``` r
# Number of customers per city
p1 = data %>%
  group_by(City_Category) %>%
  summarize(total = n_distinct(User_ID)) %>%
  ggplot(aes(x = City_Category, y = total, fill = City_Category)) +
  geom_bar(stat = "identity") +
  xlab("") +
  ylab("Numer of Customers") +
  theme.hc() +
  guides(fill = FALSE)

# Total purchase per city
p2 = data %>%
  group_by(City_Category) %>%
  summarize(purchases = sum(Purchase)) %>%
  ggplot(aes(x = City_Category, y = purchases, fill = City_Category)) +
  geom_bar(stat = "identity") +
  xlab("") +
  ylab("Total Purchase") +
  scale_y_continuous(labels = dollar) +
  theme.hc() +
  guides(fill = FALSE)

# Arrange the plots
grid.arrange(p1, p2, nrow = 1, top = "Customer and Purchase Amount by City Category")
```

<img src="files/unnamed-chunk-14-1.png" style="display: block; margin: auto;" />

Although city C has the largest population, the citizens of B are spending more. This could indicate that B is the wealthiest city of the three given its higher average spending.

### Occupation Distribution

``` r
# Occupation Distribution
p1 = data %>%
  group_by(Occupation) %>%
  summarize(total = n_distinct(User_ID)) %>%
  ggplot(aes(x = Occupation, y = total, fill = Occupation)) +
  geom_bar(stat = "identity") +
  ggtitle("Number of Customers by Occupation") +
  ylab("") +
  theme.hc() +
  guides(fill = FALSE) +
  theme(axis.title.x = element_blank(), axis.text.x = element_blank())

# Total Purchase Ammount by Occupation
p2 = data %>%
  group_by(Occupation) %>%
  summarize(purchase = sum(as.numeric(Purchase))) %>%
  ggplot(aes(x = Occupation, y = purchase, fill = Occupation)) +
  geom_bar(stat = "identity") +
  scale_y_continuous(labels = dollar) +
  ggtitle("Total Purchase Amount by Occupation") +
  ylab("") +
  theme.hc() +
  guides(fill = FALSE) +
  theme(axis.title.x = element_blank(), axis.text.x = element_blank())

# Average Purchase Ammount by Occupation
p3 = data %>%
  group_by(Occupation) %>%
  summarize(purchase = sum(as.numeric(Purchase)),
            count = n_distinct(User_ID),
            average = purchase / count) %>%
  ggplot(aes(x = Occupation, y = average, fill = Occupation)) +
  geom_bar(stat = "identity") +
  scale_y_continuous(labels = dollar) +
  ggtitle("Average Purchase Amount by Occupation") +
  ylab("") +
  theme.hc() +
  guides(fill = FALSE)

# Plot the three figures and align their axes
p1 = ggplotGrob(p1)
p2 = ggplotGrob(p2)
p3 = ggplotGrob(p3)
maxWidth = grid::unit.pmax(p1$widths[2:5], p2$widths[2:5], p3$widths[2:5])
p1$widths[2:5] = as.list(maxWidth)
p2$widths[2:5] = as.list(maxWidth)
p3$widths[2:5] = as.list(maxWidth)
grid.arrange(p1, p2, p3, ncol = 1, top = "")
```

<img src="files/unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

As we can see the total purchase amount by occupation is primary due to the customer distribution among occupatins. This becomes clearer when looking at the average purchase per occupations, where the differences are much smaller.

Product Analysis
----------------

What can we find out about our products?

### How many unique products are there?

``` r
data %>% distinct(Product_ID) %>% nrow()
```

    [1] 3623

### Distribution of Products in the Main Categories

``` r
data %>%
  group_by(Product_Category_1) %>%
  summarise(n = n_distinct(Product_ID)) %>%
  ggplot(aes(x = Product_Category_1, y = n, fill = Product_Category_1)) +
  geom_bar(stat = "identity") +
  ggtitle("Distribution of Products in the Main Categories") +
  ylab("Number of unique Products") +
  xlab("") +
  theme.hc() +
  guides(fill = FALSE)
```

<img src="files/unnamed-chunk-17-1.png" style="display: block; margin: auto;" />

We can clearly see that the product categories 1, 5 and 8 have the largest number of unique products.

### Sales and Price Range by Category

``` r
p1 = data %>%
  group_by(Product_Category_1) %>%
  summarize(count = n()) %>%
  ggplot(aes(x = Product_Category_1, y = count, fill = Product_Category_1)) +
  geom_bar(stat = "identity") +
  ylab("Transactions") +
  xlab("") +
  theme.hc() +
  guides(fill = FALSE)

p2 = data %>%
  ggplot(aes(x = Product_Category_1, y = Purchase, fill = Product_Category_1)) +
  geom_boxplot() +
  scale_y_continuous(labels = dollar) +
  ylab("Purchase Price") +
  xlab("Main Product Categroy") +
  theme.hc() +
  guides(fill = FALSE)

# Plot the three figures and align their axes
p1 = ggplotGrob(p1)
p2 = ggplotGrob(p2)
maxWidth = grid::unit.pmax(p1$widths[2:5], p2$widths[2:5], p3$widths[2:5])
p1$widths[2:5] = as.list(maxWidth)
p2$widths[2:5] = as.list(maxWidth)
grid.arrange(p1, p2, ncol = 1, top = "Sales and Price Range per Category")
```

<img src="files/unnamed-chunk-18-1.png" style="display: block; margin: auto;" />

In the plot on the top we see the total transactions by main product category, below the chart shows the price range of the products for each of the main product categories. The products that were purchased most frequently are in the low to medium price range.

### Pareto Check: Does the 80/20 Rule apply?

According to the famous [Pareto priciple](https://en.wikipedia.org/wiki/Pareto_principle) or 80/20 Rule, in many cases 80% of the results come from only 20% of the efforts. This principle can be observed in many situations, however here we want to check if in our data 20% of the products generate 80% of revenue. The idea is that if we can find these 20% of products, we can better focus and prioritize our efforts.

First, let's create a product ranking by their individual contribution to the revenue and accumulate the percentage.

``` r
# List of products sorted from best seller to worst seller  
product.ranking = data %>%
  group_by(Product_ID) %>%
  summarize(purchase = sum(Purchase)) %>%
  arrange(desc(purchase)) %>%
  mutate(percent = purchase / sum(Purchase),
         cum.percent = cumsum(percent)) 

head(product.ranking)
```

    # A tibble: 6 x 4
      Product_ID purchase percent cum.percent
      <chr>         <dbl>   <dbl>       <dbl>
    1 P00025442   275324. 0.00549     0.00549
    2 P00110742   263826. 0.00526     0.0107 
    3 P00255842   246524. 0.00491     0.0157 
    4 P00184942   240609. 0.00480     0.0205 
    5 P00059442   239483. 0.00477     0.0252 
    6 P00112142   238826. 0.00476     0.0300 

Now plot the number of products vs cummulative contribution to revenue.

``` r
# Determine number of the products that are responsible for 80% of the revenue
n.top.products = sum(product.ranking$cum.percent <= 0.8)

# Plot Pareto line
product.ranking %>%
  ggplot(aes(x = seq(1, n_distinct(Product_ID), 1), y = cum.percent)) +
  geom_path() +
  geom_vline(xintercept = n.top.products, linetype = "dashed", color = "#619CFF") +
  geom_hline(yintercept = 0.8, linetype = "dashed", color = "#619CFF") +
  ggtitle("Pareto Principle (80/20 Rule)") +
  xlab("Number of Products") +
  ylab("% of Purchase (cummulative)") +
  scale_y_continuous(labels = percent, breaks = seq(0, 1, 0.1)) +
  scale_x_continuous(breaks = c(seq(500, 3500, 1000), n.top.products)) +
  theme.hc()
```

<img src="files/unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

It appears that the top 25.6% (927) of products are responsible for 80% of the revenue. Therefore, we can say that the Pareto principle approximately applies to our data. The first (927) products of the previously established ranking deserve special attention when we want to optimize the product portfolio or marketing activities.

Predicting Customer Spending
============================

Now we want to see if we can predict how much a customer will spend based on their demographic data.

Each row in the data represents one transaction. Since one customer may purchase multiple products, the same `User_ID` can appear multiple times in the data. Therefore, we first need to group the data so that each row represents one customer.

``` r
user.data = data %>%
  group_by(User_ID, Gender, Age, Occupation, City_Category, Stay_In_Current_City_Years, Marital_Status) %>%
  summarize(total.purchase = sum(Purchase))
  
head(user.data)
```

    # A tibble: 6 x 8
    # Groups:   User_ID, Gender, Age, Occupation, City_Category,
    #   Stay_In_Current_City_Years [6]
      User_ID Gender Age   Occupation City_Category Stay_In_Current~
      <chr>   <fct>  <fct> <fct>      <fct>         <fct>           
    1 1000001 F      0-17  10         A             2               
    2 1000002 M      55+   16         C             4+              
    3 1000003 M      26-35 15         A             3               
    4 1000004 M      46-50 7          B             2               
    5 1000005 M      26-35 20         A             1               
    6 1000006 F      51-55 9          A             1               
    # ... with 2 more variables: Marital_Status <fct>, total.purchase <dbl>

Before we start building models, let's check if we have outliers in the response variable `total.purchase` that we should take care of.

``` r
box1 = user.data %>% 
  ggplot(aes(x = "", total.purchase)) + 
  geom_boxplot(fill = "#619CFF") +
  xlab("") +
  ylab("Total Purchase per User") +
  theme.hc()
box1
```

<img src="files/unnamed-chunk-22-1.png" style="display: block; margin: auto;" />

It appears that we have some significant outliers. In order to not distort the results we will replace the outliers with the median.

``` r
# Function to impute outliers
impute.outliers = function(x, removeNA = TRUE){
  quantiles = quantile(x, c(0.1, 0.9), na.rm = removeNA)
  x[x < quantiles[1]] = median(x, na.rm = removeNA)
  x[x > quantiles[2]] = median(x, na.rm = removeNA)
  x
}

# Apply function to response variable
imputed.data = impute.outliers(user.data$total.purchase)

# Create new boxplot
box2 = user.data %>%
  add_column(total.purchase.imputed = imputed.data) %>%
  ggplot(aes(x = "", y = total.purchase.imputed)) + 
  geom_boxplot(fill = "#00BA38") +
  xlab("") +
  ylab("Total Purchase per User") +
  theme.hc()

# Arrange the plots
grid.arrange(box1, box2, nrow = 1, top = "Distr. of Response Var. Before and After Cleaning Outliers")
```

<img src="files/unnamed-chunk-23-1.png" style="display: block; margin: auto;" />

Now that we got rid of the extreme outliers, the new data looks a lot better. However, the response variable still contains some relatively large outliers. To get a more realistic idea of the models' performance, we should therefore not only look at the Root Mean Squared Error (RMSE) but also at the Mean Absolute Error (MAE). Since the MAE does not penalize large errors as heavily as RMSE it is more robust to outliers.

Next we will replace cleaned values for `total.purchase` in our dataset and create the training and test sets.

``` r
# Replace the outliers in the data
user.data = user.data %>%
  select(-c("total.purchase")) %>%
  add_column(total.purchase = as.numeric(imputed.data)) 

# Create training and test sets
set.seed(2019)
index = sample(x = c(T, F), size = nrow(user.data), replace = T, prob = c(0.8, 0.2))
user.data.train = user.data[index,-1]
user.data.test = user.data[!index,-1]

# Preview the training data
head(user.data.train)
```

    # A tibble: 6 x 7
      Gender Age   Occupation City_Category Stay_In_Current~ Marital_Status
      <fct>  <fct> <fct>      <fct>         <fct>            <fct>         
    1 F      0-17  10         A             2                0             
    2 M      55+   16         C             4+               0             
    3 M      26-35 15         A             3                0             
    4 M      46-50 7          B             2                1             
    5 M      26-35 20         A             1                1             
    6 F      51-55 9          A             1                0             
    # ... with 1 more variable: total.purchase <dbl>

Let's compare two fundamentally different approaches: Regression and a tree-based model.

First, we will fit a Linear Model.

``` r
# Fit linear model
lm = lm(total.purchase ~ ., data = user.data.train)

# Get predictions
pred.lm = predict(lm, user.data.test)

# Get RMSE
rmse.lm = sqrt(mean((pred.lm - user.data.test$total.purchase)^2))

# Get MAE
mae.lm = mean(abs(pred.lm - user.data.test$total.purchase))

# Print the results
tibble(RMSE = rmse.lm,
       MAE = mae.lm)
```

    # A tibble: 1 x 2
       RMSE   MAE
      <dbl> <dbl>
    1 4183. 3263.

As expected, the MAE is significantly lower than the RMSE.

Now we will create a Random Forest to see if it fits the data better.

``` r
rf = randomForest(total.purchase ~., 
                  data = user.data.train,
                  ntree = 1000
                  )

# Get predictions
pred = predict(rf, user.data.test)

# Get RMSE
rmse.rf = sqrt(mean((pred - user.data.test$total.purchase)^2))

# Get MAE
mae.rf = mean(abs(pred - user.data.test$total.purchase))

# Print the results
tibble(RMSE = rmse.rf,
       MAE = mae.rf)
```

    # A tibble: 1 x 2
       RMSE   MAE
      <dbl> <dbl>
    1 4214. 3283.

The Random Forest performed slightly worse than the Linear Model.

To further improve the predictions we could perform a grid search to find the optimal hyperparameters and try other models such as Gradient Boosting or a Neural Network. However, this would be out of the scope of this project.

Recommendation System
=====================

In this section we want to create a [Recommendation system](https://en.wikipedia.org/wiki/Recommender_system), which produces a list of product recommendations for a customer. Essentially, the way this works is by predicting which products customers will like based on their similarity to other customers. Here the employed measure of similarity is the [Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index). To determine the similarity of two customers, the number of products both users bought is divided by the total number of products bought by customer A or customer B. To find the best model we will build a model based on the similarities between customers (User Based Collaborative Filtering (UBCF)) and another model based on the similarities between products (Item Based Collaborative Filtering (IBCF)), and compare the two methods.

To start, we first need to create a binary rating matrix with `User_ID`s in the rows and `Product_ID`s in the columns.

``` r
# Create binary rating matrix
brm = data %>% 
  group_by(User_ID, Product_ID) %>%
  summarize(count = n()) %>%
  spread(key = Product_ID, value = count) %>%
  replace(., is.na(.), 0) %>%
  as.data.frame() %>%
  column_to_rownames("User_ID") %>%
  as.matrix() %>%
  as("binaryRatingMatrix")

# Show a snapshot of the matrix
as(brm, "matrix")[1:8,1:6]
```

            P00000142 P00000242 P00000342 P00000442 P00000542 P00000642
    1000001      TRUE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000002     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000003     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000004     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000005     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000006      TRUE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000007     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE
    1000008     FALSE     FALSE     FALSE      TRUE     FALSE     FALSE

Now we create the training and test set including only users who have made a sufficient number of purchases.

``` r
# Include only users that have bought at least 20 products
brm = brm[rowCounts(brm) > 20,]  

# Create training and test set
set.seed(2019)
index = sample(x = c(T,F), size = nrow(brm), replace = T, prob = c(0.8, 0.2))
brm.train = brm[index,]
brm.test = brm[-index,]
```

Let's build the recommendation model using IBCF and glimpse the list of product recommendations. Given that our data is binary, we will use Jaccard as a measure of similarity as mentioned earlier.

``` r
# Train the model
ibcf = Recommender(data = brm.train,
                   method = "IBCF", 
                   parameter = list(method = "Jaccard")) 

# Make predictions
ibcf.pred = predict(object = ibcf,
                    newdata = brm.test,  # generate recommendations on test data
                    n = 5)               # number of recommendations

# Assign correct product names to predicted indices
ibcf.rec.matrix = sapply(ibcf.pred@items, function(x) colnames(brm)[x])

# Have a look at the recommendations for the first 5 users
as.tibble(ibcf.rec.matrix)[,1:5]
```

    # A tibble: 5 x 5
      `1000002` `1000003` `1000005` `1000006` `1000008`
      <chr>     <chr>     <chr>     <chr>     <chr>    
    1 P00270942 P00057642 P00270942 P00025442 P00114942
    2 P00127742 P00270942 P00271142 P00110742 P00110742
    3 P00114942 P00025442 P00057642 P00114942 P00025442
    4 P00025442 P00114942 P00044442 P00193542 P00044442
    5 P00125942 P00145042 P00058042 P00336342 P00184942

So far so good, it seems that it is working.

In order to determine the optimal model, we will use 5-fold cross-validation. Check the validation splits.

``` r
# Create validation splits
set.seed(2019)
eval.sets = evaluationScheme(data = brm,
                             method = "cross-validation",
                             k = 5, # number of folds
                             given = 20
                             )

# Print size of the 5 validation folds
sapply(eval.sets@runsTrain, length)
```

    [1] 3792 3792 3792 3792 3792

Now let's not only use IBCF, but also UBCF to see how the two compare. To check if our recommendations have any value at all we include a model that generates random recommendations as well. Then we will plot the precision and recall scores for different numbers of product recommendations.

In the context of this data, precision and recall mean the following:

-   **Precision:** Of the all recommended products, how many are actually relevant to the customer?
-   **Recall:** Of all the products that are relevant to the customer, what is the fraction that the algorithm recommended?

``` r
# Define the models that we are going to compare
models = list(IBCF = list(name = "IBCF", params = list(method = "Jaccard")),
              UBCF = list(name = "UBCF", params = list(method = "Jaccard")),
              RAND = list(name = "RANDOM", params = NULL))

# Create the models 
results = evaluate(x = eval.sets,
                   method = models,
                   n = c(1:5, seq(10,100,10)) # Try different numbers of product recommendations
                   )
```

``` r
# Plot precision and recall values for the different numbers of recommended products
plot(results, "prec/rec", annotate = 2, legend = "topright") + 
  title("Precision vs Recall for Different Numbers of Recommendations")
```

<img src="files/unnamed-chunk-32-1.png" style="display: block; margin: auto;" />

    integer(0)

We can clearly see that both IBCF and UBCF performed a lot better than the random recommendations. The best model is UBCF. We can also see that the more products we recommend, the higher our recall value, which makes sense because as we recommend more products we should expect that the likelihood that we recommend a relevant product increases. As recall increases, precision decreases because the more products we recommend the more products the products the user will see, which are not relevant. Given this trade-off and the fact that we do not want to spam the customers with product recommendations, 5 seems to be a reasonable value giving us a precision of about 0.3.
