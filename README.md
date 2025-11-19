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

## Model Selection 
We have a stationary dataset. Incredible. It is time to select the model to be used on the dataset. I am already leaning towards ARIMA, but need to determine the value of the coefficients by looking at the ACF and pACF plots derived from the transformed data. Those plots are included below: 
<img width="1312" height="528" alt="image" src="https://github.com/user-attachments/assets/f4b93ca7-62ce-4ac8-9e7e-9e5fcc333d9d" />

The ACF has a big negative spike at lag 1, indicative of an MA(1) process. The pACF shows strong negative spikes at lag 1, with other less prominent spikes featured along the lags, matching an AR(1)/AR(2) process, though not as strongly as the ACF matches the MA(1) process. No seasonality needs to be included in the model, as the dataset does not have one. 

I decided to try three models on the data, based on the above diagnostics. 

### ARIMA(0,1,1) 
Generally the standard model to try first, which is what I did. 
Terrible model for this dataset for the following reasons: 
- MA(1) = -0.999, which means there is a ton of noise in the dataset and ARIMA cannot reliably estimate the model structure. A lot of spikes, burtsts, volatility and structural breaks are the cause of this.
- Year to year volatility is prominent in the data
- Q = 148.92 (p=0.00), meaning the residuals are not white noise + autocorrelation was not removed
- JB = 4563 (p=0.00), meaning the residuals are extremely non-normal w/ heavy kurtosis (extreme outliers/spikes yearly)
- H = 0.17 (p=0.00), meaning its variance is changing over time

In other words, this data requires a model that can handle volatility/shock. enter GARCH. 

### GARCH(1,1) 
This is a model used to analyze and forecast the volatility of time series data. It is apparently used a lot for finance data, which is a good sign in my opinion. Model was run with mean set to 0 as the differencing has already been done. 
Not a terrible model, but not ideal either, for the following reasons:

- α + β ≈ 0.9999, which means volatility shocks decay extremely slowly. The process is essentially near-integrated, meaning the variance is almost non-stationary.
- Volatility is extremely persistent, consistent with long periods of elevated uncertainty in the dataset.
- Residuals still show bursts and occasional spikes, meaning the variance model does not fully capture the violent shock structure in the data.
- Standardized residuals still show heavy tails, implying that a normal-distribution GARCH cannot handle the extreme shocks present in the series.

Which brings me to the ARIMA + GARCH combination.

### ARIMA(0,1,1) + GARCH(1,1) 
This model creates residuals using the ARIMA model, then feeds those residuals into the GARCH model for modelling that accounts for residuals. The output for this model looks quite promising: 
- PArameters are stable, variance converged, model is posed well.
-  α + β ≈ 1 which is a common trend in data with a lot of volatility and shock. Both highly significant

The model is performing similarly to other GARCH models related to volatile data (based on a few of my searches). But the residuals must be plotted + interpreted to be certain. 

<img width="560" height="435" alt="image" src="https://github.com/user-attachments/assets/6a6b6cc3-a592-439c-9b48-82e51429bb89" />

This plot shows: 
- no obvious autocorrelation
- Relatively constant spread
- range of 2 to -2 with a few spikes
- Centred around 0
- no visible clustering (meaning the volatility is accounted for)
- The conditional volatility displays long run structural variance growth, which again, is typical for data like this.

The next step is to use this model to forecast some data. 


## Forecasting 

