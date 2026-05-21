### Regression
> MSE, MAE, Huber

points to consider:
- Noise distribution of the labels. 
  - MSE assumes gaussian noise. (b'coz, it is MLE for a Gaussian Likelyhood)
  - heavy tailed label/outliers can become dominant in case of mse
  - Huber is the industry standard for any real world sensor data in regression
- cost of asymemtry of errors
  - Standard losses are symmetric.
  - example: cost of predicting the depth 5m when true is 3m vs 3m when actually 5m.
  - understimating or overstimating might have different consequence and would depend on the task and goal
  - pick a asymmetric loss or bias the post prediction logic.
  - assymetric can be fundamental or context dependent.
- Scale invariant or absolute.
  - whether the loss changed with the scale or not.

### Classification
> BCE/CE/Focal

points to consider:
- class distribution stationary?
  - does the data and its distribution chage with time in the production setup or not.
  - Loss-level fix: class-balanced loss, focal loss. 
  - Data-level fix: oversample minority. 
  - Inference-level fix: threshold tuning. 
  - All three are legitimate — the question is where the abstraction boundary sits in your system.
- 