# Lessons Learned
A collection of handy code snippets from my time at HICC, where I worked with Canadian demographic and housing data. 

## Arrow makes quick work of larger than memory datasets

The [Arrow R Package](https://arrow.apache.org/docs/r/) uses the Apache Arrow platform to help us work with larger than memory data. R usually performs computations in RAM (your computer's short term memory), but this can pose a problem for larger datasets I needed to work with at HICC that were larger than the size of my computer's entire RAM (16 GB). Arrow moves computations onto disk (your computer's long term memory) to avoid this. Used in conjunction with the [parquetize](https://cran.r-project.org/web/packages/parquetize/index.html) package, it's a dream for dealing with a lot of information without need for cloud resources. Here are a few functions of note:

[`open_dataset()`](https://arrow.apache.org/docs/r/reference/open_dataset.html), which opens a connection to a parquetized and partitioned dataset. This creates an Arrow Dataset, which is listed as an `<Object containing active binding>`. Arrow datasets are not located in memory, but we can pass them dplyr functions to work with them like a normal data frame. Displaying it's contents gives us the object's size and a list of it's columns and their types. 

``` r
Table
11009816 rows x 26 columns
$col1 <int32>
$col2 <string>
$col3 <string>
$col4 <string>
...
$col42 <string>
```

[`schema()`](https://arrow.apache.org/docs/11.0/r/reference/Schema.html), which is used to define the variables in the dataset and their type (integer64, string, etc). Denoting variable types can help to save memory. 

[`compute()`](https://dplyr.tidyverse.org/reference/compute.html) which is used to force computation of a database query. Apply `compute()` to save the results of a query. It will be stored in a temporary table remotely. 

[`collect()`](https://dplyr.tidyverse.org/reference/compute.html) retrieves data into a local tibble. Be wary to not run `collect()` on the entire large dataset, as loading it all at once can cause your R session to crash. Instead, use `collect()` to inspect the beginning or end of the table as below:

``` r
df <- open_dataset(source_data, schema = schema)

# Use collect on summary statics table
df %>%
group_by(var1, var2) %>%
summarize(mean = mean(var1)) %>%
collect()

# Use collect on head
df %>% 
head(500) %>%
collect()

```

`?acero` can be used to check compatible dplyr verbs. 

At times, we need to use dplyr functions that aren't supported by arrow, and in this case we use `to_duckdb` to swap our arrow connection to [DuckBD](https://duckdb.org/docs/api/r.html) and later swap back to arrow. More on this [here](https://duckdb.org/2021/12/03/duck-arrow.html). 


``` r
df <- df_arrow %>%
  
  # swap to duckdb to use distinct with .keep_all = TRUE
  to_duckdb() %>%
  
  # keep only rows with distinct vars 
  dictinct(var1, var2, var3, .keep_all = TRUE) %>%
  
  # change connection back to arrow
  to_arrow

```

## Parameterized reporting with Quarto shapes up your data for sharing

It's no secret that I'm a big fan of Quarto ([see my website for example](alburycatalina.github.io)). Parametrized reporting is a new ball game because it allows for the creation of customizable reports based on varying datasets and user inputs. By defining parameters, I can generate tailored outputs based on certain criteria, making it easy to analyze different scenarios without manually editing the code. It's flexible, reproducible, and efficient. And to top it all off, it also works in both R and Python!

[R Ladies' Jadey Ryan gives a great introduction to using quarto for Parameterized reporting in her talk hosted on Youtube.](https://www.youtube.com/watch?v=MKjz_xkMgxY). Here are some highlights: 

Set the `params` argument in the front matter .qmd file as below. 

``` r
---
title: "Document"                   # Metadata
format:                             # Set format types
  html: default                                     
  docx: default                           
params:                             # Set default parameter key-value pairs
  community: "Vancouver"                                
---
    
Report content goes here.           # Write content


```

Then, you can call on your parameter using the dplyr `$` notation within the qmd file. 

``` r
`params$community`

[1] "Vancouver"
```

Run the following to render a single document while testing. Using the `execute_params` argument, the parameter variable can be changed. 

``` r 
quarto_render(input = "quarto-template.qmd", # template
output_format = "docx", # document format 
execute_params = list(community = "Toronto") # name of community
```

To iterate a list of parameters generating multiple documents, I created a wrapper script that relied on `quarto_render` and `purrr`'s `pwalk`. 

``` r
 reports <- df |> 
  mutate(output_format = "docx",
         output_file = paste("prepopulated_doc",
                             community_name,
                             "report.docx", 
                             sep = "-"),
         execute_params = map(community_name, \(community_name) list(community_name = community_name))) |> 
  select(output_format, output_file, execute_params)

pwalk(
  .l = reports,
  .f = quarto_render,
  input = "HNA-Template_en.qmd",
  .progress = TRUE
)
```

## The `cansim` and `cmhc` packages serve up fresh Canadian demographic data to go

The prolific [mountainmath](https://doodles.mountainmath.ca/) has developed a suite of packages that allow for accessing the heaps of publicly available Canadian demographic data. Of note are the `cansim` and `cmhc` packages. 

`cansim` allows for interfacing with the Statistics Canada NDM that hosts information updated on the daily, monthly, and occasional frequency. 

`get_cansim` retrives a data table based on NDM number. 

For larger tables, the `get_cansim_sqlite` function stores the data for later use in a SQLite database locally. 

`dplyr` verbs can be used on the objects generated by both of these, which is extremely handy. 



