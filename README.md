# Condo Unit Forecasting
Work related project involving the forecasting of monthly "units" created. 

## Business Need 
- The ability to predict the total number of units created every month, to a certain degree of accuracy.

## Data Collection 
All the data used for this project exists within the Condominium Authority of Ontario's databases, as a result, they cannot be shared publicly. I used an SQL query to fetch the following for each corporate entry in our : 
- Account Name
- Registration Date
- Total Units

## Pre-Processing 
This specific dataset is already quite clean within our database, as a result, not much of that was required. 

## Visualization 
The first step in working with time series data is to visualize it - and that has been done below: 
<img width="1634" height="528" alt="image" src="https://github.com/user-attachments/assets/7e599104-a29b-48ee-85eb-dc713ff19ae9" />

Many of the first points are zeros, which will affect most models. As a result, I have removed all points with a date earlier than January 1 1967. Leading to a new graph: 
<img width="1634" height="528" alt="image" src="https://github.com/user-attachments/assets/956bbb7d-a7cf-49b7-bbfc-8a4145b9d28c" />

The data is then resampled (monthly sums), in order to create a cleaner graph (and because I want to be able to make monthly predictions). I have also included the moving average and standard deviation to make conclusions about the nature of the data. 
<img width="1634" height="547" alt="image" src="https://github.com/user-attachments/assets/2d9a17a8-7edc-4e90-ab78-f439458c1df0" />

It is clear that the variance and standard deviation of the dataset are not constant (i.e, the dataset is non-stationary). This means that certain transformations must be applied before we can continue with the forecasting process. To verify this, I ran an Augumented Dickey Fuller (ADF) test, to the following results: 

```
ADF: -2.242311654766631
p: 0.191248565295295
Critical Values: {'1%': np.float64(-3.439960610754265), '5%': np.float64(-2.8657809735786244), '10%': np.float64(-2.5690284373908066)}
The dataset is not stationary
```
The resampled data does not show a seasonal trend, but for good measure I decided to plot the seasonality to be certain: 
<img width="1626" height="547" alt="image" src="https://github.com/user-attachments/assets/c50300e2-78fe-4b4d-b08a-63e77cbd0f86" />

So it appears we have a highly volatile dataset, very skewed to the right (data points get bigger as time goes further), with no clear seasonality trends. 

## Transformations 
I applied a few transformations to the dataset. The transformations and their reasonings are listed below: 
- Log transform: needed to stabilize the variance of the dataset
- Differencing: done to the transformed data, used to stabilize the mean/remove any trends from the dataset.

An ADF test done on the transformed data provided the following results: 

```
ADF: -4.38551811759864
p: 0.0003148483108971253
Critical Values: {'1%': np.float64(-3.4400894360545475), '5%': np.float64(-2.865837730028723), '10%': np.float64(-2.5690586760471605)}
The dataset is stationary
```


