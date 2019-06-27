```r
library(rsofun)
library(dplyr)
library(readr)
library(lubridate)
source("get_obs_bysite_fluxnet2015.R")
source("train_predict_fvar.R")
source("predict_nn.R")
source("prepare_trainingdata_fvar.R")
source("remove_outliers.R")
source("get_consecutive.R")
source("add_alpha.R")
```


# Get FLUXNET 2015 data

Read observational data including soil moisture for one site (`"AU-How"`) from a FLUXNET 2015 data file (standard).
```r
getvars <- c( 
  "GPP_NT_VUT_REF", 
  "GPP_DT_VUT_REF",                
  "LE_F_MDS", 
  "LE_F_MDS_QC", 
  "NEE_VUT_REF_NIGHT_QC", 
  "NEE_VUT_REF_DAY_QC", # quality flag that goes with GPP 
  "TA_F",
  "VPD_F",
  "P_F",
  "SW_IN_F",
  "NETRAD"
  )
df <- get_obs_bysite_fluxnet2015( sitename="AU-How", path_fluxnet2015="./", timescale="d", getvars=getvars )

## get some mo' (modelled soil moisture)

# load("/alphadata01/bstocker/nn_fluxnet2015/data/modobs_fluxnet2015_s11_s12_s13_with_SWC_v3.Rdata")
# df_wcont_splash  <- fluxnet[[ "AU-How" ]]$ddf$s11 %>% 
#                       as_tibble() %>% 
#                       mutate( date=ymd( paste0( as.character(year), "-01-01") ) + days(doy-1), wcont_splash=wcont )
# df_wcont_et_orth %>% write_csv( path = "inst/ext/df_wcont_et_orth.csv")
df_wcont_et_orth <- read_csv( file = "inst/ext/df_wcont_et_orth.csv")

# df_wcont_swbm    <- fluxnet[[ "AU-How" ]]$ddf$s12 %>% 
#                       as_tibble() %>% 
#                       mutate( date=ymd( paste0( as.character(year), "-01-01") ) + days(doy-1), wcont_swbm=wcont )
# df_wcont_et %>% write_csv( path = "inst/ext/df_wcont_et.csv")
df_wcont_et <- read_csv( file = "inst/ext/df_wcont_et.csv")

# df_wcont_et      <- fluxnet[[ "AU-How" ]]$ddf$swc_by_etobs %>% 
#                       as_tibble() %>% 
#                       mutate( date=ymd( paste0(as.character(year), "-01-01") ) + months(moy-1) + days(dom-1), wcont_et=soilm_from_et )
# df_wcont_swbm %>% write_csv( path = "inst/ext/df_wcont_swbm.csv")
df_wcont_swbm <- read_csv( file = "inst/ext/df_wcont_swbm.csv")

# df_wcont_et_orth <- fluxnet[[ "AU-How" ]]$ddf$swc_by_etobs %>% 
#                       as_tibble() %>% 
#                       mutate( date=ymd( paste0(as.character(year), "-01-01") ) + months(moy-1) + days(dom-1), wcont_et_orth=soilm_from_et_orthbucket )
# df_wcont_splash %>% write_csv( path = "inst/ext/df_wcont_splash.csv")
df_wcont_splash <- read_csv( file = "inst/ext/df_wcont_splash.csv")

```

# Prepare data for training

Prepare training data, removing NAs and outliers.
```r
## use all observational soil moisture data
varnams_soilm <- df %>% 
  dplyr::select( starts_with("SWC_") ) %>% 
  dplyr::select( -ends_with("QC") ) %>% 
  names()

## define settings used by multiple functions as a list
settings <- list( 
  target        = "transp_obs", 
  predictors    = c("temp","vpd","swin"), 
  varnams_soilm = varnams_soilm 
  )

df_train <- prepare_trainingdata_fvar( df, settings )
```

# Get soil moisture threshold

NOT YET IMPLEMENTED
```r
soilm_threshold_obs <- profile_soilmthreshold_fvar(
  df_train,
  settings,
  hidden_good        = 10, 
  hidden_all         = 10, 
  nrep               = 3, 
  weights            = NA, 
  package            = "nnet" 
 )
```

# Train models and predict `fvar`

Train models for one set of soil moisture input data. In this case it's observational data.
```r
df_nn_soilm_obs <- train_predict_fvar( 
  df_train,
  settings,
  soilm_threshold    = 0.6, # hard coded here instead of using output from profile_soilmthreshold_fvar()
  hidden_good        = 10, 
  hidden_all         = 10, 
  nrep               = 3, 
  weights            = NA, 
  package            = "nnet" 
  )
```

Do the training again using another set of soil moisture data from a model.
```r
df <- df %>% left_join( select(df_wcont_swbm, date, wcont_swbm), by="date" )
settings$varnams_soilm <- "wcont_swbm"

df_train <- prepare_trainingdata_fvar( df, settings )

df_nn_soilm_swbm <- train_predict_fvar( 
  df_train,
  settings,
  hidden_good        = 10, 
  hidden_all         = 10, 
  soilm_threshold    = 0.6, 
  nrep               = 3, 
  weights            = NA, 
  package            = "nnet" 
  )
```

# Identify soil moisture droughts

```r
## first aggregate fvar derived from different soil moisture datasets
df_nn <- bind_rows( df_nn_soilm_obs, df_nn_soilm_swbm ) %>% 
  group_by( date ) %>%
  summarise( 
    fvar       = mean( fvar, na.rm=TRUE ), 
    fvar_min   = min(  fvar, na.rm=TRUE ),
    fvar_max   = max(  fvar, na.rm=TRUE ), 
    fvar_med   = median(  fvar, na.rm=TRUE ), 
    var_nn_pot = mean( var_nn_pot, na.rm=TRUE ), 
    var_nn_act = mean( var_nn_act, na.rm=TRUE )
    ) %>%
  left_join( select_( df, "date", settings$target ), by = "date" )

out_droughts <- get_droughts_fvar( 
  df_nn, 
  nam_target     = settings$target, 
  leng_threshold = 10, 
  df_soilm       = select(df_wcont_swbm, date, soilm=wcont_swbm),
  df_par         = select(df, date, par=ppfd),
  par_runmed     = 10
  )
```

Plot what we got.
```r
with( out_droughts$df_nn, plot(date, fvar_smooth_filled, type="l" ) )
rect( 
  out_droughts$df_nn$date[out_droughts$droughts$idx_start], rep( -99, nrow(out_droughts$droughts) ), 
  out_droughts$df_nn$date[(out_droughts$droughts$idx_start+out_droughts$droughts$len-1)], rep( 99, nrow(out_droughts$droughts) ), 
  col=rgb(0,0,0,0.2), border=NA )
```

# Align data by droughts

```r
dovars <- c("fvar", "soilm")
df_nn <- out_droughts$df_nn %>% mutate(site="AU-How")
df_alg <- align_events( 
  select( df_nn, site, date, one_of(dovars)), 
  select( df_nn, site, date, isevent=is_drought_byvar_recalc ),
  dovars,
  leng_threshold = 10,
  before=20,
  after=79,
  nbins=10,
  do_norm=FALSE,
  normbin=2 )
```


Plot what we got
```r
with( df_alg$df_dday_aggbydday, plot(    dday, fvar_median, type="l", ylim=c(0,1.2)) )
with( df_alg$df_dday_aggbydday, polygon( c(dday, rev(dday)), c(fvar_q33, rev(fvar_q66)), col=rgb(0,0,0,0.2), border = NA ) )
abline(h=1, lty=3)
abline(v=0, col='red')
```