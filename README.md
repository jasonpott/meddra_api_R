# MEDDRA API with R

Collection of functions to work with the MEDDRA.org api to code and classify adverse events in clinical trials.

While working with this API, I was unable to find any example code to work with the API in R so publishing this in order to hopefully save others the headache.

What these functions offer is a mechanism to construct a valid json submission to the api. Other functions required that a json file be constructed externally and sent.

## Dependancies
The functions used here make use of functions from:
- httr
- jsonlite
- dplyr

The [dmtools](https://github.com/KonstantinRyabov/dmtools/tree/62d753239cf54eb6beee8da02606477f9b447130) package for R was initially used to interact with the MEDDRA API but these functions did not sufficiently address the submission requirements for the current MEDDRA api which requies a json query structure. The authentication functions in this repo reproduce the underlying authentication methods from dmtools.

## Functions:


### API status

This function does not require authentication.

```r
meddra_api_status <- function(){
  api_status <- content(GET(url = "https://mapisbx.meddra.org/api/status",
                            add_headers(accept = "*/*")),
                        "text")

  print(api_status, quote = FALSE )
}

```

### Obtain MEDRA authentication token

This function requires that the used has a registration with the MEDDRA service and has obtained an API key

```r
meddra_obtain_token <- function(){

  url <- 'https://mid.meddra.org/connect/token'

  meddra_id <- rstudioapi::askForPassword(prompt = "Please provide MEDRA USER ID")

  api_key <- rstudioapi::askForPassword(prompt = "Please provide MEDRA API Key")

  # token <- meddra_auth(token_url, meddra_id, api_key)

  # The below code is adapted from the authentication functions in dmtools
  token <- httr::POST(
    url = url,
    body = list(grant_type = "password", username = meddra_id, password = api_key, scope = "meddraapi"),
    httr::authenticate("mspclient", "clientsecret"),
    encode = "form"
  )
token <- httr::content(token)$access_token
  return(token)
}

token <- meddra_obtain_token()
```

### Obtain term level of MEDDRA code

This function requires that the token has been obtained using the meddra_obtain_token function above and assigned to token

```r

check_meddra_term_level <- function(medra_code) {
  term_level <- suppressMessages(
    httr::content(httr::GET(
      url = paste(
        "https://mapisbx.meddra.org/api/hist",
        medra_code,
        1,
        "English",
        "Release",
        sep = "/"
      ),
      httr::add_headers(
        "accept" = "application/json",
        "Content-Type" = "text/json",
        "Authorization" = paste("Bearer", token)
      )
    ),
    "text") %>%
      jsonlite::fromJSON() %>%
      # obtaining the term level from the latest version of MEDDRA
      dplyr::arrange(desc(medDraVer)) %>%
      dplyr::slice(1) %>%
      dplyr::select(termLevel)
  )
  return(term_level$termLevel)
}
```

### Obtain term label from MEDDRA API

This function needs to have a logical evaluation of the term level implemented using the check_meddra_term_level function above to correctly create the json model submission

```r

 json_output <- list(
      bview = unbox("SOC"),
      rsview = unbox("Release"),
      code = unbox(0),
      pcode = unbox(0),
      syncode = unbox(0),
      lltcode = unbox(0),
      ptcode = unbox(medra_code),
      hltcode = unbox(0),
      hlgtcode = unbox(0),
      soccode = unbox(0),
      smqcode = unbox(0),
      type = unbox("PT"),
      addlangs = list(),
      rtype = unbox("M"),
      lang = unbox("English"),
      ver = unbox(20.1),
      kana = unbox(FALSE),
      separator = unbox(0)
    ) %>% jsonlite::toJSON()

   url <- "https://mapisbx.meddra.org/api/type"

  # Submit the call to the MEDDRA API
  response <- httr::POST(
    url,
    httr::add_headers(
      "accept" = "application/json",
      "Content-Type" = "text/json",
      "Authorization" = paste("Bearer", token)
    ),
    body = json_output
  )
# Take the returned json convert to df
  search_result <- httr::content(response, "text") %>%
    jsonlite::fromJSON() %>%
    as.data.frame()%>%
    # Some pt terms have multiple SOC this returns the primary allocation
    dplyr::filter(primarysocfg == "Y")
  # obtain the ptname for the medra code
  label <- search_result$ptname

  return(label)

}
```


### Obtain term SOC from MEDDRA API

This function repeats the label methods but extracts and returns the SOC this could be encorporated into a function argument

```r
meddra_obtain_soc <- function(medra_code){

  json_output <- list(
    bview = unbox("SOC"),
    rsview = unbox("Release"),
    code = unbox(0),
    pcode = unbox(0),
    syncode = unbox(0),
    lltcode = unbox(0),
    ptcode = unbox(medra_code),
    hltcode = unbox(0),
    hlgtcode = unbox(0),
    soccode = unbox(0),
    smqcode = unbox(0),
    type = unbox("PT"),
    addlangs = list(),
    rtype = unbox("M"),
    lang = unbox("English"),
    ver = unbox(20.1),
    kana = unbox(FALSE),
    separator = unbox(0)
  ) %>% jsonlite::toJSON()

  url <- "https://mapisbx.meddra.org/api/type"

  # Submit the call to the MEDDRA API
  response <- httr::POST(
    url,
    httr::add_headers(
      "accept" = "application/json",
      "Content-Type" = "text/json",
      "Authorization" = paste("Bearer", token)
    ),
    body = json_output
  )
  # Take the returned json convert to df
  search_result <- httr::content(response, "text") %>%
    jsonlite::fromJSON() %>%
    as.data.frame() %>%
    # Some pt terms have multiple SOC this returns the primary allocation
    dplyr::filter(primarysocfg == "Y")
  # obtain the soc name for the meddra code
  label <- search_result$socname

  return(label)

}
```
