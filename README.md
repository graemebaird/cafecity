# Cafecity

Cafecity is a data aggregation, modeling, and forecasting tool for estimating high-growth areas for cafe placement. The underlying model calibration and workflow is extensible to any combination of businesses/locations/areas, but in this instance the specific application is on predicting the growth of coffee shops on a neighborhood-basis in the city of Seattle. 

Scraping and geospatial wrangling are handled in R, model fitting and geoJSON generation are handled in Python, and the interface is constructed with JS/HTML/CSS, primarily using Leaflet.js to serve output from Flask. 

Underlying tech: 
1. Python (pandas, shapely, flask, numpy, geopandas, pickle, pyGAM)
2. R (rgdal, dplyr, magrittr, reshape2, raster, Quandl, rgeos, tidyverse, httr, tidyr, geosphere)
3. Javascript (Leaflet, D3, DC, JQuery, Keen, Queue, Crossfilter, Underscore)
4. HTML/CSS (Bootstrap, Keen)
5. Nginx/Gunicorn (on AWS server)

## Data sources

Data for the Seattle implementation are collected from several sources: 

1. The United States Census Bureau, from `https://lehd.ces.census.gov/data/` and `https://www.census.gov/geo/maps-data/data/tiger-geodatabases.html`, for tract shapes, demographic and employment information

2. The Quandl API, from `https://www.quandl.com/tools/api`, for real estate pricing and trends information

3. Seattle Open Data, from `https://data.seattle.gov/`, for zoning, permits, and neighborhood shapes

4. Google Places and Yelp API, from `https://developers.google.com/places/web-service/intro` and `https://www.yelp.com/developers`, for current locations of cafes and their characteristics (data are collated and transformed to raster counts before storage)

## Data and model process

1. APIs and shapefiles are pulled from sources listed above, combined, and rasterized into a coordinate-based model matrix for analysis

2. Collinear/irrelevant features are eliminated from the model matrix using conditional random forests for feature selection

3. Spatial autocorrelative features are added to the feature space by convolving across a lat/long/cafe-count matrix with sub-matrices of varying sizes, creating n-nearest cell neighbor cafe sums

4. The modified model matrix is used to train a generalized additive model, using pyGAM's GridSearch algorithm to perform cross-validation and hyperparameter smoothing

5. The trained model is pickled to file

6. Residual estimation and prediction is performed by first taking marginal predictions independent of autocorrelative features. After the first prediction instance, autocorrelation features are re-generation using the convolve method in step 3 and repeated until predictions stabilize (10 iterations is more than enough)

7. For forecasting, step 6 is conducted on an underlying forecast of city data. This is conducted via weighted assignment of growth rates based on building permit and real estate data, which takes Puget Sound Planning Authority growth rate estimates and distributes them across the city to plausible growth hotspots. Neighborhoods with stronger weights are transformed more per time period (month), and the converse is true. 

## Underlying model

The model fit to this data is a generalized additive model with smoothing splines, described by `Y ~ Pois(lambda)` and `log(lambda) ~ β + f.(X)`, the usual GAM structure with a series of splines and a tensor product smooth for the autocorrelation features. 

## Visualization and user input

User input is processed through a python/Flask instance to produce predictions from the model object. Predictions are matched with user-specified shapes (census tract, neighborhood, etc) and piped to visualization libraries in Javascript (mostly Leaflet and DC/D3) for plotting. 

## File structure

1. `input/` - raw and processed data, excluded from git repo for size and permissions issues
2. `static/` - frontend custom and generic css/js libraries
3. `source/` - underlying data process files (unused in server-side activities)
4. `app.py`, `templates/`, `tables.py` - Flask server files
5. `finalized_model.sav` - fitted GAM modelfile
