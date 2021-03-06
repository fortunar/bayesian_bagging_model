# About the package
The purpose of **matchForecast** package is to forecast sport match outcomes by performing bagging using two level modeling. The approach takes into account the uncertainty in the inputs to avoid overconfidence of ML models. Since the inputs to a match outcome prediction problem are often uncertain the approach can enhance the predictive performance of ML models.

# Which problems is it applicable to?
The package is applicable to problems where the attributes are uncertain (estimated). In other words, attributes of feature vectors are estimated based on some known prior observations (measurements). 

The use case will be described on the problem of forecasting basketball games. Prior to a match we want to forecast we have data of the team performance in past matches. These data include 2 point shots made, 3 point shots made, rebounds etc. For a new basketball match, we do not know how a team will perform, which is why we somehow need to estimate the attributes of the actual feature vectors that we feed into a ML algorithm. We might be tempted to take the mean values of performance data from past matches. However we obtain point estimates of the attributes which contain uncertainty. When we don't have a lot of prior games available mean estimates can lead to overconfidence and poor performance. This package models the data from past games with  distributions. Multiple predictive models are then bagged by training ML models on feature vectors obtained from fitted distributions.

Let's generalize the use cases. Problems that this package can be used for have a specific data structure of matches and objects. Our goal is to predict the outcome of new matches and we want the algorithm to learn based on prior matches. Objects (e.g. teams) have attributes and participate in matches. One or multiple objects can participate in each match. This package assumes objects have the same types of measured attributes. The attributes of participating objects in past matches are known and so is the outcome. For a new event we do not know the attributes or the outcome. We need to estimate the unknown attributes and predict the outcome.

# Input data: match-objects format

The main input to this package are matches in the form of a data frame. One match corresponds to one row in the data frame. Each match contains columns with measurements of attributes of participating objects. Each match can also contain a `TIME` attribute, which can be used to weight the importance of prior matches for estimating the distributions.

Columns of match-object formatted data frame include object ID's in the format: `ID_<number>` and attributes in the format `<attr_name>_<number>`. `<number>` suffix connects objects to attributes. The suffixed number of corresponding object and attributes should be the same. Column for the outcome (dependent variable) of the match should be denoted with `y`. 

An example of the columns of the data for basketball:


| TIME | y | ID_1 | ID_2 | P2M_1| P3M_1 | P2M_2 | P3M_2| ...|
|-------|---|--------|-------|---------|----------|----------|-----|---|


In the basketball case ID_1 refers to ID of the home team and P2M_1, P3M_1 to the attributes of the home team. ID_2 refers to the ID of away team and P2M_2, P3M_2 to its attributes. y is the outcome of the match.

## How do we model attributes?
We treat attributes of objects as random variables. Known measurements of an attribute are realizations of the corresponding random variable. For each random variable we compute multiple possible distributions using known prior measurements and a parametric assumption. Parametric assumption is our belief of how the variable is distributed. If a random variable is a count, a good guess is that it is Poisson distributed. Therefore in that case we can use the Poisson - Gamma Bayesian model to fit multiple Poisson distributions to the random variable. With supported Bayesian models we can also use priors. This package supports the following univariate Bayesian models out of the box: 

* Poisson - gamma model for count attributes
* Normal model with sample variance
* Normal - Inverse Gamma model for numeric attributes
* Bernoulli - Beta model for binary attributes

One multivariate Bayesian model is also supported: Multivariate Normal - Inverse Wishart model to jointly model numeric attributes.

User can also use custom models (see examples for more information).

## How do we use the distributions we get from modeling attributes?
The distributions we obtain from modeling attributes of objects are used to construct feature vectors that are fed into ML models. Any quantity derived from the distribution can be used: means, medians, quantiles etc. 

# Usage
## Install the package:
```R
devtools::install_github("fortunar/matchForecast")
```
Detailed documentation of all the parameters and use cases is available by installing the package by typing `?<name_of_function>`.

## Building the model - match_forecast_model()
```R
# Function for obtaining a single instance of a ML model. In this case it builds a 
# logistic regression model on data obtained by applying a transformation 
# function (in this case "means") on the distributions fitted to attributes.
get_model <- function(data) {
  return(glm(y ~ ., family = binomial(link = "logit"), data = data))
}

# Example of a prior specification (basketball example), for an attribute P3A,
# which is a Poisson variable and measures 3-point attempts.
# Conjugate prior for Poisson distribution is the Gamma distribution.
# Parameters 'a' and 'b' in the list below are shape and rate parameters
# of gamma priors
priors_example <- list(
  "San Antonio Spurs" =
    list(
      "P3A" = list(a = 241.2, b = 12.4),
      ...
    ),
  "Golden State Warriors" =
    list(
      "P3A" = list(a = 280.9, b = 13.2),
      ...
    ),
  ...
)

# Builds the Match Forecast Model
mf_model <- match_forecast_model(
  # Data frame in match-object format
  data = data_train,
  # Parametric assumption about attributes - in this case we assume
  # all attributes are distributed by Poisson distribution
  input_model_specification = 'poisson',
  # How many distributions to fit to each attribute (also equal
  # to the number of bagged models
  num_models = 100,
  # How to build feature vectors from distributions
  transformation = "means",
  get_model = get_model,
  priors = priors_example
)
```

## Predicting with match_forecast_model

```R
# Predict function to act on a single bagged model and a single sampled test data set obtained
# from object models by applying a transformation function (e.g. means).
predict_fun <- function(model, data) {
  data$y <- NULL
  return(predict(model, newdata = data, type = "response"))
}

# This yields 100 * 100 predictions. It returns a list of lists 
# of predictions. Each prediction (list) contains fields:
# - predictions: output of predict_fun passed to this method
# - idx_of_bagged_model: index of the bagged model (in this case 1-100)
# - idx_of_test_set: index of test set generated by first level model (in this case 1-100)
results <- predict(
  # The model of type 'match_forecast_model'
  mf_model,
  # Test data (data to predict): needs to include only participating 
  # object ID's
  data_new = data_new,
  predict_fun = predict_fun
)

```

## More examples


### Modeling independent attributes with different parametric assumptions

In the example above, we modeled all attributes using Poisson - Gamma model by passing setting  `input_model_specification =  'poisson'`. If we want to use different parametric assumptions for attributes we need to pass a list to `input_model_specification` with keys being column names and values strings representing parametric assumptions.

Parametric assumptions  (independent attributes) and their corresponding strings:

 - Poisson - Gamma model: "poisson"
 - Normal model with sample variance: "normal"
 - Normal - Inverse Gamma model: "normal_ig"
 - Bernoulli - Beta model: "bernoulli"

Example code (just building the model):
```
# List of keys being attribute names (without object number suffix!)
# and values being strings corresponding to parametric assumptions
mixed_input_models <- list(
  # 2 point shots made
  P2M = "poisson"
  # 3 point shots made
  P3M = "normal"
  # Win ratio
  WR = "bernoulli"
)
mf_model <- match_forecast_model(
    data = data_train,
    input_model_specification = mixed_input_models,
    num_models = 100,
    transformation = "means",
    get_model = get_model
  )
```

### Modeling dependent attributes with multivariate normal - inverse Wishart model

Multivariate normal - inverse Wishart model jointly models all attributes of an object.

Example code:

```
bbm <- match_forecast_model(
    data = data,
    input_model_specification = list(dependent = T, type = "mvnormal_iw"),
    num_models = 100,
    transformation = "means",
    get_model = get_model
)
```
## Advanced usage - custom models

Packages uses two levels of representation for object models. An object model is a model of one single object (e.g. a team). It contains attribute models which model attributes of the object. Attribute models can be univariate or multivariate.

Object model structure is defined by the package. In order to make a custom model, one needs to create an instance of 'object_model' and fill it with custom attribute models.

### Building a custom model

To write a custom model, a function needs to be passed in as the `input_model_specification`. As arguments it receives data from a single object and `num_models` parameter. It should return a list of length `num_models` of instances of class 'object_model'. One 'object_model' represents a single model fit to the object. It contains parameter `models`, which is a list of key-value pairs. Keys correspond to attributes and values to corresponding attribute models (e.g. of type "poisson").

Constructor for the class 'object_model':
```
object_model <- function() {
  return(
    structure(
      list(
        models = list()
      ),
      class = "object_model"
    )
  )
}
# Methods defined for the class 'object_model'
add_model <- function(obj, name, model)
{
  UseMethod("add_model", obj)
}

# Adds an attribute model with a specific name to the models attribute of 'object_model'
add_model.object_model <- function(obj_model, name, model) {
  obj_model$models[[name]] <- model
  return(obj_model)
}


# Get a vector of means of the attribute models
mean.object_model <- function(obj_model) {
  return(lapply(obj_model$models, mean))
}

# Get a vector representing a sample from the attribute models
sample.object_model <- function(obj_model) {
  return(lapply(inst_model$models, sample))
}

# Get vector of variances of the attribute models
var.object_model <- function(obj_model) {
  return(lapply(obj_model$models, var))
}
```

To write a custom model one needs to create an instance of class 'object_model' and fill its `models` parameter with custom attribute models. Each of the attribute models inside the `models` parameter should support `mean`, `var` and `sample` methods. For an example on how to write an attribute model refer to the source code of existing attribute models (Poisson - Gamma model, Normal model with sample variance etc.).

Example code for the Poisson - Gamma attribute model (already supported by the package):

```
# data parameter is a numerical vector of known attribute values
input_model_poisson <- function(data, num_models, prior) {
  if (num_models == 1) {
    return(
      list(
        structure(
          list(
            rate = sum(data)/length(data)
          ),
          class = "poisson"
        )
      )
    )
  }
  prior_a <- 0
  prior_b <- 0
  if (!is.null(prior)) {
    prior_a <- prior$a
    prior_b <- prior$b
  }
  rates <- rgamma(
    num_models,
    shape = sum(data) + prior_a,
    rate = length(data) + prior_b
  )
  fits <- rep(
    list(
      structure(
        list(
          rate = NULL
        ),
        class = "poisson"
      )
    ),
    length(rates)
  )
  for (i in 1:length(rates)) {
    fits[[i]]$rate <- rates[[i]]
  }
  return(fits)
}

mean.poisson <- function(model) {
  return(model$rate)
}

sample.poisson <- function(model, num_samples) {
  return(rpois(num_samples, model$rate))
}

var.poisson <- function(model) {
  return(model$rate)
}
```