
# Attribute data operations {#attr}

## Prerequisites {-}

- This chapter requires the following packages to be installed and attached: 


```r
library(sf)
library(raster)
library(tidyverse)
```

- It also relies on **spData**, which loads datasets used in the code examples of this chapter:


```r
library(spData)
```

## Introduction

Attribute data is non-spatial information associated with geographic (geometry) data.
A bus stop provides a simple example: its position would typically be represented by latitude and longitude coordinates (geometry data), in addition to its name.
The name is an *attribute* of the feature (to use Simple Features terminology) that bears no relation to its geometry.
<!-- idea: add an example of a bus stop (or modify a previous example so it represents a bus stop) in the previous chapter  -->

Another example is the elevation value (attribute) for a specific grid cell in raster data.
Unlike vector data, the raster data model stores the coordinate of the grid cell only indirectly:
There is a less clear distinction between attribute and spatial information in raster data.
Say, we are in the 3^rd^ row and the 4^th^ column of a raster matrix.
To derive the corresponding coordinate, we have to move from the origin three cells in x-direction and four cells in y-direction with the cell resolution defining the distance for each x- and y-step.
The raster header gives the matrix a spatial dimension which we need when plotting the raster or when we want to combine two rasters, think, for instance, of adding the values of one raster to another (see also next chapter).
<!-- should we somewhere add a table comparing advantages/disadvantages of using the vector or raster data model, would fit nicely into chapter 2 -->

The focus of this chapter is manipulating geographic objects based on attributes such as the name of a bus stop and elevation.
For vector data this means operations such as subsetting and aggregation (see sections \@ref(vector-attribute-subsetting) and \@ref(vector-attribute-aggregation)).
These non-spatial operations have spatial equivalents:
the `[` operator in base R, for example, works equally for subsetting objects based on their attribute and spatial objects, as we will see in Chapter \@ref(spatial-operations).
This is good news: skills developed here are cross-transferable, meaning that this chapter lays the foundation for Chapter \@ref(spatial-operations), which extends the methods presented here to the spatial world.
Sections \@ref(vector-attribute-joining) and \@ref(vec-attr-creation) demonstrate how to join data onto simple feature objects using a shared ID and how to create new variables, respectively.

Raster attribute data operations are covered in Section \@ref(manipulating-raster-objects), which covers creating continuous and categorical raster layers and extracting cell values from one layer and multiple layers (raster subsetting). 
Section \@ref(summarizing-raster-objects) provides an overview of 'global' raster operations which can be used to characterize entire raster datasets.

## Vector attribute manipulation

Geographic vector data in R are well-support by `sf`, a class which extends the `data.frame`.
Thus `sf` objects have one column per attribute variable (such as 'name') and one row per observation, or *feature* (e.g. per bus station).
`sf` objects also have a special column to contain geometry data, usually named `geometry`.
The `geometry` column is special because it is a *list-colum*, which can contain multiple geographic entities (points, lines, polygons) per row.
In Chapter \@ref(spatial-class) we saw how to perform *generic methods* such as `plot()` and `summary()` on `sf` objects.
**sf** also provides methods that allow `sf` objects to behave like regular data frames:


```r
methods(class = "sf") # methods for sf objects, first 12 shown
```


```r
#>  [1] aggregate             cbind                 coerce               
#>  [4] initialize            merge                 plot                 
#>  [7] print                 rbind                 [                    
#> [10] [[<-                  $<-                   show                 
```



Many of these functions, including `rbind()` (for binding rows of data together) and `$<-` (for creating new columns) were developed for data frames.
A key feature of `sf` objects is that they store spatial and non-spatial data in the same way, as columns in a `data.frame` (the geometry column is typically called `geometry`).

\BeginKnitrBlock{rmdnote}<div class="rmdnote">The geometry column of `sf` objects is typically called `geometry` but any name can be used.
The following command, for example, creates a geometry column named g:
  
`st_sf(data.frame(n = world$name_long), g = world$geom)`

This enables geometries imported from spatial databases to have a variety of names such as `wkb_geometry` and `the_geom`.</div>\EndKnitrBlock{rmdnote}

`sf` objects also support `tibble` and `tbl` classes used in the tidyverse, allowing 'tidy' data analysis workflows for spatial data.
Thus **sf** enables the full power of R's data analysis capabilities to be unleashed on geographic data.
Before using these capabilities it's worth re-capping how to discover the basic properties of vector data objects.
Let's start by using base R functions to get a measure of the `world` dataset:


```r
dim(world) # it is a 2 dimensional object, with rows and columns
#> [1] 177  11
nrow(world) # how many rows?
#> [1] 177
ncol(world) # how many columns?
#> [1] 11
```

Our dataset contains ten non-geographic columns (and one geometry list-column) with almost 200 rows representing the world's countries.

Extracting the attribute data of an `sf` object is the same as removing its geometry:


```r
world_df = st_set_geometry(world, NULL)
class(world_df)
#> [1] "data.frame"
```

This can be useful if the geometry column causes problems, e.g. by occupying large amounts of RAM, or to focus the attention on the attribute data.
For most cases, however, there is no harm in keeping the geometry column because non-spatial data operations on `sf` objects only change an object's geometry when appropriate (e.g. by dissolving borders between adjacent polygons following aggregation).
This means that proficiency with attribute data in `sf` objects equates to proficiency with data frames in R.
For many applications, the tidyverse package **dplyr** offers the most effective and intuitive approach of working with data frames, hence the focus on this approach in this section.^[
Unlike objects of class `Spatial*` of the **sp** package, `sf` objects are also compatible with the **tidyverse** packages **dplyr** and **ggplot2**.
The former provides fast and powerful functions for data manipulation 
<!-- (see [Section 6.7](https://csgillespie.github.io/efficientR/data-carpentry.html#data-processing-with-data.table) of @gillespie_efficient_2016),  -->
and the latter provides powerful plotting capabilities.
]

### Vector attribute subsetting

Base R subsetting functions include `[`, `subset()` and  `$`.
**dplyr** subsetting functions include `select()`, `filter()`, and `pull()`.
Both sets of functions preserve the spatial components of attribute data in `sf` objects.

The `[` operator can subset both rows and columns. 
You use indices to specify the elements you wish to extract from an object, e.g. `object[i, j]`, with `i` and `j` typically being numbers or logical vectors --- `TRUE`s and `FALSE`s --- representing rows and columns (they can also be character strings, indicating row or column names).
<!-- you can also use `[`(world, 1:6, 1) -->
Leaving `i` or `j` empty returns all rows or columns, so `world[1:5, ]` returns the first five rows and all columns.
The examples below demonstrate subsetting with base R.
The results are not shown; check the results on your own computer:


```r
world[1:6, ] # subset rows by position
world[, 1:3] # subset columns by position
world[, c("name_long", "lifeExp")] # subset columns by name
```

A demonstration of the utility of using `logical` vectors for subsetting is shown in the code chunk below.
This creates a new object, `small_countries`, containing nations whose surface area is smaller than 10,000 km^2^:


```r
sel_area = world$area_km2 < 10000
summary(sel_area) # a logical vector
#>    Mode   FALSE    TRUE 
#> logical     170       7
small_countries = world[sel_area, ]
```

The intermediary `sel_object` is a logical vector that shows that only seven countries match the query.
A more concise command, that omits the intermediary object, generates the same result:


```r
small_countries = world[world$area_km2 < 10000, ]
```

The base R function `subset()` provides yet another way to achieve the same result:


```r
small_countries = subset(world, area_km2 < 10000)
```

<!-- , after the package has been loaded: [or - it is a part of tidyverse] -->
Base R functions are mature and widely used.
However, the more recent **dplyr** approach has several advantages.
It enables intuitive workflows.
It is fast, due to its C++ backend.
This is especially useful when working with big data as well as **dplyr**'s database integration.
The main **dplyr** subsetting functions are `select()`, `slice()`, `filter()` and `pull()`.

<div class="rmdnote">
<p><strong>raster</strong> and <strong>dplyr</strong> packages have a function called <code>select()</code>. When using both packages, the function in the most recently attached package will be used, ‘masking’ the incumbent function. This can generate error messages containing text like: <code>unable to find an inherited method for function ‘select’ for signature ‘&quot;sf&quot;’</code>. To avoid this error message, and prevent ambiguity, we use the long-form function name, prefixed by the package name and two colons (usually omitted from R scripts for concise code): <code>dplyr::select()</code>.</p>
</div>

`select()` selects columns by name or position.
For example, you could select only two columns, `name_long` and `pop`, with the following command (note the sticky `geom` column remains):


```r
world1 = dplyr::select(world, name_long, pop)
names(world1)
#> [1] "name_long" "pop"       "geom"
```

`select()` also allows subsetting of a range of columns with the help of the `:` operator: 


```r
# all columns between name_long and pop (inclusive)
world2 = dplyr::select(world, name_long:pop)
```

Omit specific columns with the `-` operator:


```r
# all columns except subregion and area_km2 (inclusive)
world3 = dplyr::select(world, -subregion, -area_km2)
```

Conveniently, `select()` lets you subset and rename columns at the same time, for example:


```r
world4 = dplyr::select(world, name_long, population = pop)
names(world4)
#> [1] "name_long"  "population" "geom"
```

This is more concise than the base R equivalent:


```r
world5 = world[, c("name_long", "pop")] # subset columns by name
names(world5)[names(world5) == "pop"] = "population" # rename column manually
```

`select()` also works with 'helper functions' for advanced subsetting operations, including `contains()`, `starts_with()` and `num_range()` (see the help page with `?select` for details).

All **dplyr** functions including `select()` always return a dataframe-like object. 
To extract a single vector, one has to explicitly use the `pull()` command.
The subsetting operator in base R (see `?[`), by contrast, tries to return objects in the lowest possible dimension.
This means selecting a single column returns a vector in base R.
To turn off this behavior, set the `drop` argument to `FALSE`.


```r
# create throw-away dataframe
d = data.frame(pop = 1:10, area = 1:10)
# return dataframe object when selecting a single column
d[, "pop", drop = FALSE]
select(d, pop)
# return a vector when selecting a single column
d[, "pop"]
pull(d, pop)
```

Due to the sticky geometry column, selecting a single attribute from an sf-object with the help of `[()` returns also a dataframe.
Contrastingly, `pull()` and `$` will give back a vector.


```r
# dataframe object
world[, "pop"]
# vector objects
world$pop
pull(world, pop)
```

`slice()` is the row-equivalent of `select()`.
The following code chunk, for example, selects the 3^rd^ to 5^th^ rows:


```r
slice(world, 3:5)
```

`filter()` is **dplyr**'s equivalent of base R's `subset()` function.
It keeps only rows matching given criteria, e.g. only countries with a very high average of life expectancy:


```r
# Countries with a life expectancy longer than 82 years
world6 = filter(world, lifeExp > 82)
```

The standard set of comparison operators can be used in the `filter()` function, as illustrated in Table \@ref(tab:operators): 


Table: (\#tab:operators)Table of comparison operators that result in boolean (TRUE/FALSE) outputs.

Symbol      Name                            
----------  --------------------------------
`==`        Equal to                        
`!=`        Not equal to                    
`>, <`      Greater/Less than               
`>=, <=`    Greater/Less than or equal      
`&, |, !`   Logical operators: And, Or, Not 

<!-- describe these: ==, !=, >, >=, <, <=, &, | -->
<!-- add warning about = vs == -->
<!-- add info about combination of &, |, ! -->

**dplyr** works well with the ['pipe'](http://r4ds.had.co.nz/pipes.html) operator `%>%`, which takes its name from the Unix pipe `|` [@grolemund_r_2016].
It enables expressive code: the output of a previous becomes the first argument of the next function, enabling *chaining*.
This is illustrated below, in which the `world` dataset is subset by columns (`name_long` and `continent`) and the first five rows (result not shown).


```r
world7 = world %>%
  filter(continent == "Asia") %>%
  dplyr::select(name_long, continent) %>%
  slice(1:5)
```

The above chunk shows how the pipe operator allows commands to be written in a clear order:
the above run from top to bottom (line-by-line) and left to right.
The alternative to `%>%` is nested function calls, which is harder to read:


```r
world8 = slice(
  dplyr::select(
    filter(world, continent == "Asia"),
    name_long, continent),
  1:5)
```

<!-- I commented-this out as it's overly wordy (RL) -->
<!-- This generates the same result --- verify this with `identical(world7, world8)` --- in the same number of lines of code, but in a much more confusing way, starting with the function that is called last! -->

<!-- There are additional advantages of pipes from a communication perspective: they encourage adding comments to self-contained functions and allow single lines *commented-out* without breaking the code. -->

### Vector attribute aggregation

Aggregation operations summarize datasets by a 'grouping variable', typically an attribute column (spatial aggregation is covered in the next chapter).
An example of attribute aggregation is calculating the number of people per continent based on country-level data (one row per country).
The `world` dataset contains the necessary ingredients: the columns `pop` and `continent`, the population and the grouping variable respectively.
The aim is to find the `sum()` of country populations for each continent.
This can be done with the base R function `aggregate()` as follows:


```r
world_agg1 = aggregate(pop ~ continent, FUN = sum, data = world, na.rm = TRUE)
class(world_agg1)
#> [1] "data.frame"
```

The result is a non-spatial data frame with six rows, one per continent, and two columns reporting the name and population of each continent (see Table \@ref(tab:continents) with results for the top 3 most populous continents).

`aggregate()` is a generic function which means that it behaves differently depending on its inputs.
**sf** provides a function that can be called directly with `sf:::aggregate()` that is activated when a `by` argument is provided, rather than using the `~` to refer to the grouping variable:


```r
world_agg2 = aggregate(world["pop"], by = list(world$continent),
                       FUN = sum, na.rm = TRUE)
class(world_agg2)
#> [1] "sf"         "data.frame"
```

As illustrated above, an object of class `sf` is returned this time.
`world_agg2` which is a spatial object containing 6 polygons representing the columns of the world.

`summarize()` is the **dplyr** equivalent of `aggregate()`.
It usually follows `group_by()`, which specifies the grouping variable, as illustrated below:


```r
world_agg3 = world %>%
  group_by(continent) %>%
  summarize(pop = sum(pop, na.rm = TRUE))
```

This approach is flexible and gives control over the new column names.
This is illustrated below: the command calculates the Earth's population (~7 billion) and number of countries (result not shown):


```r
world %>% 
  summarize(pop = sum(pop, na.rm = TRUE), n = n())
```

In the previous code chunk `pop` and `n` are column names in the result.
`sum()` and `n()` were the aggregating functions.
The result is an `sf` object with a single row representing the world (this works thanks to the geometric operation 'union', as explained in section \@ref(geometry-unions)).

Let's combine what we've learned so far about **dplyr** by chaining together functions to find the world's 3 most populous continents (with `dplyr::n()` ) and the number of countries they contain (the result of this command is presented in Table \@ref(tab:continents)):


```r
world %>% 
  dplyr::select(pop, continent) %>% 
  group_by(continent) %>% 
  summarize(pop = sum(pop, na.rm = TRUE), n_countries = n()) %>% 
  top_n(n = 3, wt = pop) %>%
  st_set_geometry(value = NULL) 
```


Table: (\#tab:continents)The top 3 most populous continents, and the number of countries in each.

continent           pop   n_countries
----------  -----------  ------------
Africa       1154946633            51
Asia         4311408059            47
Europe        669036256            39

\BeginKnitrBlock{rmdnote}<div class="rmdnote">More details are provided in the help pages (which can be accessed via `?summarize` and `vignette(package = "dplyr")` and Chapter 5 of [R for Data Science](http://r4ds.had.co.nz/transform.html#grouped-summaries-with-summarize). </div>\EndKnitrBlock{rmdnote}

<!-- `sf` objects are well-integrated with the **tidyverse**, as illustrated by the fact that the aggregated objects preserve the geometry of the original `world` object. -->
<!-- Here, we even had to make some efforts to prevent a spatial operation. -->
<!-- When `aggregate()`ing the population we have just used the population vector.  -->
<!-- Had we used the spatial object (world[, "population"]), `aggregate()` would have done a spatial aggregation of the polygon data.  -->
<!-- The same would have happened, had we not dismissed the geometry prior to using the `summarize()` function. -->
<!-- We will explain this so-called 'dissolving polygons' in more detail in the the next chapter. -->

<!-- Todo (optional): add exercise exploring similarities/differences with `world_continents`? -->

<!-- should it stay or should it go (?) aka should we present the arrange function?: -->
<!-- Jannes: I would suggest to leave the arrange function as an exercise to the reader. -->

<!-- ```{r} -->
<!-- # sort variables -->
<!-- ## by name -->
<!-- world_continents %>%  -->
<!--   arrange(continent) -->
<!-- ## by population (in descending order) -->
<!-- world_continents %>%  -->
<!--   arrange(-pop) -->
<!-- ``` -->

###  Vector attribute joining

<!-- https://github.com/dgrtwo/fuzzyjoin -->
<!-- http://r4ds.had.co.nz/relational-data.html -->
<!-- non-unique keys -->

Combining data from different sources is a common task in data preparation. 
Joins do this by combining tables based on a shared 'key' variable.
**dplyr** has multiple join functions including `left_join()` and `inner_join()` --- see `vignette("two-table")` for a full list.
These function names follow conventions used in the database language [SQL](http://r4ds.had.co.nz/relational-data.html) [@grolemund_r_2016, Chapter 13]; using them to join non spatial datasets to `sf` objects is the focus of this section.
**dplyr** join functions work the same on data frames and `sf` objects, the only important difference being the `geometry` list column.
The result of data joins can be either an `sf` or `data.frame` object.
The most common type of attribute join on spatial data takes an `sf` object as the first argument and adds columns to it from a `data.frame` specified as the second argument.

To illustrate the utility of joins, imagine that you are researching global coffee production.
You have managed to extract data hidden-away in a PDF document supplied by the International Coffee Organization (ICO).
The results are stored in a data frame called `coffee_data` which has 3 columns:
one (`name_long`) containing the names of coffee-producing nations and the other two reporting coffee production statistics for 2016 and 2017 (see `?coffee_data` for further information).
We will use a 'left join' (meaning the left-hand dataset is kept intact) to merge this dataset with the pre-existing `world` dataset.
The result of the code chunk below is a new `sf` object.


```r
world_coffee = left_join(world, coffee_data)
#> Joining, by = "name_long"
class(world_coffee)
#> [1] "sf"         "data.frame"
```

The resulting simple features object is the same as the orignal `world` object but has two new variables (with column indeces 11 and 12) reporting coffee production by year.
This can be plotted as a map, as illustrated in Figure \@ref(fig:coffeemap), generated with the `plot()` function below:


```r
names(world_coffee)
#>  [1] "iso_a2"                 "name_long"             
#>  [3] "continent"              "region_un"             
#>  [5] "subregion"              "type"                  
#>  [7] "area_km2"               "pop"                   
#>  [9] "lifeExp"                "gdpPercap"             
#> [11] "coffee_production_2016" "coffee_production_2017"
#> [13] "geom"
plot(world_coffee["coffee_production_2017"])
```

<div class="figure" style="text-align: center">
<img src="figures/coffeemap-1.png" alt="World coffee production (thousand 60 kg bags) by country, 2017. Source: International Coffee Organization." width="576" />
<p class="caption">(\#fig:coffeemap)World coffee production (thousand 60 kg bags) by country, 2017. Source: International Coffee Organization.</p>
</div>

For joining to work a 'key variable' must be supplied in both datasets.
By default **dplyr** uses all variables with matching names.
In this case both `world_coffee` and `world` objects contained a variable called `name_long`, explaining the message `Joining, by = "name_long"`.
In the majority of cases where variable names are not the same you have two options:

1. Rename the key variable in one of the objects so they match.
2. Use the `by` argument to specify the joining variables.

The latter approach is demonstrated below on a renamed version of `coffee_data`:


```r
coffee_renamed = rename(coffee_data, nm = name_long)
world_coffee2 = left_join(world, coffee_renamed, by = c(name_long = "nm"))
```



Note that the name in the original object is kept, meaning that `world_coffee` and the new object `world_coffee2` are identical.
Another feature of the result is that it has the same number of rows as the original dataset.
Although there are only 47 rows of data in `coffee_data` all 177 the country records are kept intact in `world_coffee` and `world_coffee2`:
rows in the original dataset with no match are assigned `NA` values for the new coffee production variables.
What if we only want to keep countries that have a match in the key variable?
In that case an inner join can be used:


```r
world_coffee_inner = inner_join(world, coffee_data)
#> Joining, by = "name_long"
nrow(world_coffee_inner)
#> [1] 45
```

Note that the result of `inner_join()` has only 45 rows compared with 47 in `coffee_data`.
What happened to the remaining rows?
We can identify the rows that did not match using the `setdiff()` function as follows:


```r
setdiff(coffee_data$name_long, world$name_long)
#> [1] "Congo, Dem. Rep. of" "Others"
```

The result shows that `Others` accounts for one row not present in the `world` dataset and that the name of the `Democratic Republic of the Congo` accounts for the other:
it has been abbreviated, causing the join to miss it.
The following command uses string matching (regex) to confirm what `Congo, Dem. Rep. of` should be:


```r
str_subset(world$name_long, "Dem*.+Congo")
#> [1] "Democratic Republic of the Congo"
```



To fix this issue we will create a new version of `coffee_data` and update the name.
`inner_join()`ing the updated data frame returns a result with all 46 coffee producing nations:


```r
coffee_data$name_long[grepl("Congo,", coffee_data$name_long)] = 
  str_subset(world$name_long, "Dem*.+Congo")
world_coffee_match = inner_join(world, coffee_data)
#> Joining, by = "name_long"
nrow(world_coffee_match)
#> [1] 46
```

It is also possible to join in the other direction: starting with a non-spatial dataset and adding variables from a simple features object.
This is demonstrated below, which starts with the `coffee_data` object and adds variables from the original `world` dataset.
In contrast with the previous joins, the result is *not* another simple feature object, but a data frame in the form of a **tidyverse** tibble:
the output of a join tends to match its first argument:


```r
coffee_world = left_join(coffee_data, world)
#> Joining, by = "name_long"
class(coffee_world)
#> [1] "tbl_df"     "tbl"        "data.frame"
```

\BeginKnitrBlock{rmdnote}<div class="rmdnote">In most cases the geometry column is only useful in an `sf` object.
The geometry column can only be used for creating maps and spatial operations if R 'knows' it is a spatial object, defined by a spatial package such as **sf**.
Fortunately non-spatial data frames with a geometry list column (like `coffee_world`) can be coerced into an `sf` object as follows: `st_as_sf(coffee_world)`. </div>\EndKnitrBlock{rmdnote}

This section covers the majority joining use cases.
For more information we recommend @grolemund_r_2016, the [join vignette](https://geocompr.github.io/geocompkg/articles/join.html) in the **geocompkg** package that accompanies this book, and documentation of the **data.table** package.^[
**data.table** is a high-performance data processing package.
Its application to spatial data is covered in a blog post hosted at r-spatial.org/r/2017/11/13/perp-performance.html.
]
Another type of join is a spatial join, covered in the next chapter (section \@ref(spatial-joining)).

### Creating attributes and removing spatial information {#vec-attr-creation}
<!-- lubridate? -->

Often, we would like to create a new column based on already existing columns.
For example, we want to calculate population density for each country.
For this we need to divide a population column, here `pop`, by an area column , here `area_km2` with unit area in square km.
Using base R, we can type:


```r
world_new = world # do not overwrite our original data
world_new$pop_dens = world_new$pop / world_new$area_km2
```

Alternatively, we can use one of **dplyr** functions - `mutate()` or `transmute()`.
`mutate()` adds new columns at the penultimate position in the `sf` object (the last one is reserved for the geometry):


```r
world %>% 
  mutate(pop_dens = pop / area_km2)
```

The difference between `mutate()` and `transmute()` is that the latter skips all other existing columns (except for the sticky geometry column):


```r
world %>% 
  transmute(pop_dens = pop / area_km2)
```

`unite()` pastes together existing columns. 
For example, we want to combine the `continent` and `region_un` columns into a new column named `con_reg`.
Additionally, we can define a separator (here: a colon `:`) which defines how the values of the input columns should be joined, and if the original columns should be removed (here: `TRUE`):


```r
world_unite = world %>%
  unite("con_reg", continent:region_un, sep = ":", remove = TRUE)
```

The `separate()` function does the opposite of `unite()`: it splits one column into multiple columns using either a regular expression or character positions.


```r
world_separate = world_unite %>% 
  separate(con_reg, c("continent", "region_un"), sep = ":")
```



The two functions `rename()` and `set_names()` are useful for renaming columns.
The first replaces an old name with a new one.
The following command, for example, renames the lengthy `name_long` column to simply `name`:


```r
world %>% 
  rename(name = name_long)
```

`set_names()` changes all column names at once, and requires a character vector with a name matching each column.
This is illustrated below, which outputs the same `world` object, but with very short names: 




```r
new_names = c("i", "n", "c", "r", "s", "t", "a", "p", "l", "gP", "geom")
world %>% 
  set_names(new_names)
```

It is important to note that attribute data operations preserve the geometry of the simple features.
As mentioned at the outset of the chapter, it can be useful to remove the geometry.
To do this, you have to explicitly remove it because `sf` explicitly makes the geometry column sticky.
This behavior ensures that data frame operations do not accidentally remove the geometry column.
Hence, an approach such as `select(world, -geom)` will be unsuccessful and you should instead use `st_set_geometry()`.^[
`st_geometry(world_st) = NULL` also works to remove the geometry from `world` but overwrites the original object.
]


```r
world_data = world %>% st_set_geometry(NULL)
class(world_data)
#> [1] "data.frame"
```

## Manipulating raster objects

In contrast to the vector data model underlying simple features (which represents points, lines and polygons as discrete entities in space), raster data represent continuous surfaces.
This section shows how raster objects work, by creating them *from scratch*, building on section \@ref(an-introduction-to-raster).
Because of their unique structure, subsetting and other operations on raster datasets work in a different way, as demonstrated in section \@ref(raster-subsetting).

The following code recreates the raster dataset used in section \@ref(raster-classes), the result of which is illustrated in Figure \@ref(fig:cont-raster).
This demonstrates how the `raster()` function works to create an example raster named `elev` (representing elevations).


```r
elev = raster(nrows = 6, ncols = 6, res = 0.5,
              xmn = -1.5, xmx = 1.5, ymn = -1.5, ymx = 1.5,
              vals = 1:36)
```

The result is a raster object with 6 rows and 6 columns (specified by the `nrow` and `ncol` arguments), and a minimum and maximum spatial extent in x and y direction (`xmn`, `xmx`, `ymn`, `ymax`).
The `vals` argument sets the values that each cell contains: numeric data ranging from 1 to 36 in this case.
Raster objects can also contain categorical values of class `logical` or `factor` variables in R.
The following code creates a raster representing grain sizes (Figure \@ref(fig:cont-raster)):


```r
grain_order = c("clay", "silt", "sand")
grain_char = sample(grain_order, 36, replace = TRUE)
grain_fact = factor(grain_char, levels = grain_order)
grain = raster(nrows = 6, ncols = 6, res = 0.5, 
               xmn = -1.5, xmx = 1.5, ymn = -1.5, ymx = 1.5,
               vals = grain_fact)
```



\BeginKnitrBlock{rmdnote}<div class="rmdnote">`raster` objects can contain values of class `numeric`, `integer`, `logical` or `factor`, but not `character`.
To use character values they must first be converted into an appropriate class, for example using the function `factor()`. 
The `levels` argument was used in the preceding code chunk to create an ordered factor:
clay < silt < sand in terms of grain size.
See the [Data structures](http://adv-r.had.co.nz/Data-structures.html) chapter of @wickham_advanced_2014 for further details on classes.</div>\EndKnitrBlock{rmdnote}

`raster` objects represent categorical variables as integers, so `grain[1, 1]` returns a number that represents a unique identifier, rather than "clay", "silt" or "sand". 
The raster object stores the corresponding look-up table or "Raster Attribute Table" (RAT) as a data frame in a new slot named `attributes`, which can be viewed with `ratify(grain)` (see `?ratify()` for more information).
Use the function `levels()` for retrieving and adding new factor levels to the attribute table:


```r
levels(grain)[[1]] = cbind(levels(grain)[[1]], wetness = c("wet", "moist", "dry"))
levels(grain)
#> [[1]]
#>   ID VALUE wetness
#> 1  1  clay     wet
#> 2  2  silt   moist
#> 3  3  sand     dry
```

This behavior demonstrates that raster cells can only possess one value, an identifier which can be used to look up the attributes in the corresponding attribute table (stored in a slot named `attributes`).
This is illustrated in command below, which returns the grain size and wetness of cell IDs 1, 11 and 35, we can run:


```r
factorValues(grain, grain[c(1, 11, 35)])
#>   VALUE wetness
#> 1  sand     dry
#> 2  silt   moist
#> 3  clay     wet
```

<div class="figure" style="text-align: center">
<img src="figures/cont-raster-1.png" alt="Raster datasets with numeric (left) and categorical values (right)." width="672" />
<p class="caption">(\#fig:cont-raster)Raster datasets with numeric (left) and categorical values (right).</p>
</div>

### Raster subsetting

Raster subsetting is done with the base R operator `[`, which accepts a variety of inputs:

- row-column indexing
- cell IDs
- coordinates
- another raster object

Here, we only show the first two options since these can be considered non-spatial operations.
If we need a spatial object to subset another or the output is a spatial object, we refer to this as spatial subsetting.
Therefore, the latter two options will be shown in the next chapter (see section \@ref(spatial-raster-subsetting) in the next chapter).

The first two subsetting options are demonstrated in the commands below ---
both return the value of the top left pixel in the raster object `elev` (results not shown):


```r
# row 1, column 1
elev[1, 1]
# cell ID 1
elev[1]
```

To extract all values or complete rows, you can use `values()` and `getValues()`.
For multi-layered raster objects `stack` or `brick`, this will return the cell value(s) for each layer.
For example, `stack(elev, grain)[1]` returns a matrix with one row and two columns --- one for each layer.
<!-- In this example we have used cell ID subsetting, of course, you can also use row-column or coordinate indexing. -->
For multi-layer raster objects another way to subset is with `raster::subset()`, which extracts layers from a raster stack or brick. The `[[` and `$` operators can also be used:


```r
r_stack = stack(elev, grain)
names(r_stack) = c("elev", "grain")
# three ways to extract a layer of a stack
raster::subset(r_stack, "elev")
r_stack[["elev"]]
r_stack$elev
```

Cell values can be modified by overwriting existing values in conjunction with a subsetting operation.
The following code chunk, for example, sets the upper left cell of `elev` to 0:


```r
elev[1, 1] = 0
elev[]
#>  [1]  0  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23
#> [24] 24 25 26 27 28 29 30 31 32 33 34 35 36
```

Leaving the square brackets empty is a shortcut version of `values()` for retrieving all values of a raster.
Multiple cells can also be modified in this way:


```r
elev[1, 1:2] = 0
```

### Summarizing raster objects

**raster** contains functions for extracting descriptive statistics for entire rasters.
Printing a raster object to the console by typing its name, returns minimum and maximum values of a raster.
`summary()` provides common descriptive statistics (minimum, maximum, interquartile range and number of `NA`s).
Further summary operations such as the standard deviation (see below) or custom summary statistics can be calculated with `cellStats()`. 


```r
cellStats(elev, sd)
```

\BeginKnitrBlock{rmdnote}<div class="rmdnote">If you provide the `summary()` and `cellStats()` functions with a raster stack or brick object, they will summarize each layer separately, as can be illustrated by running: `summary(brick(elev, grain))`</div>\EndKnitrBlock{rmdnote}

Raster value statistics can be visualized in a variety of ways.
Specific functions such as `boxplot()`, `density()`, `hist()` and `pairs()` work also with raster objects, as demonstrated in the histogram created with the command below (not shown):


```r
hist(elev)
```

In case a visualization function does not work with raster objects, one can extract the raster data to be plotted with the help of `values()` or `getValues()`.

Descriptive raster statistics belong to the so-called global raster operations.
These and other typical raster processing operations are part of the map algebra scheme which are covered in the next chapter (section \@ref(map-algebra)).

<div class="rmdnote">
<p>Some function names clash between packages (e.g. <code>select</code>, as discussed in a previous note). In addition to not loading packages by referring to functions verbosely (e.g. <code>dplyr::select()</code>) another way to prevent function names clashes is by unloading the offending package with <code>detach()</code>. The following command, for example, unloads the <strong>raster</strong> package (this can also be done in the <em>package</em> tab which resides by default in the right-bottom pane in RStudio): <code>detach(&quot;package:raster&quot;, unload = TRUE, force = TRUE)</code>. The <code>force</code> argument makes sure that the package will be detached even if other packages depend on it. This, however, may lead to a restricted usability of packages depending on the detached package, and is therefore not recommended.</p>
</div>

## Exercises

For these exercises we will use the `us_states` and `us_states_df` datasets from the **spData** package:


```r
library(spData)
data(us_states)
data(us_states_df)
```

`us_states` is a spatial object (of class `sf`), containing geometry and a few attributes (including name, region, area, and population) of states within the contiguous United States.
`us_states_df` is a data frame (of class `data.frame`) containing the name and additional variables (including median income and poverty level, for years 2010 and 2015) of US states, including Alaska, Hawaii and Puerto Rico.
The data comes from the US Census Bureau, and is documented in `?us_states` and `?us_states_df`.

<!-- Attribute subsetting -->
1. Create a new object called `us_states_name` that contains only the `NAME` column from the `us_states` object. 
What is the class of the new object? <!--why there is a "sf" part? -->
1. Select columns from the `us_states` object which contain population data.
Obtain the same result using a different command (bonus: try to find three ways of obtaining the same result).
Hint: try to use helper functions, such as `contains` or `starts_with` from **dplyr** (see `?contains`).
1. Find all states with the following characteristics (bonus find *and* plot them):
    - Belong to the Midwest region.
    - Belong to the West region, have an area below 250,000 km^2^ *and* in 2015 a population greater than 5,000,000 residents (hint: you may need to use the function `units::set_units()` or `as.numeric()`).
    - Belong to the South region, had an area larger than 150,000 km^2^ or a total population in 2015 larger than 7,000,000 residents.
<!-- Attribute aggregation -->
1. What was the total population in 2015 in the `us_states` dataset?
What was the minimum and maximum total population in 2015?
1. How many states are there in each region?
1. What was the minimum and maximum total population in 2015 in each region?
What was the total population in 2015 in each region?
<!-- Attribute joining -->
1. Add variables from `us_states_df` to `us_states`, and create a new object called `us_states_stats`.
What function did you use and why?
Which variable is the key in both datasets?
What is the class of the new object?
1. `us_states_df` has two more variables than `us_states`.
How can you find them? (hint: try to use the `dplyr::anti_join` function)
<!-- Attribute creation -->
1. What was the population density in 2015 in each state?
What was the population density in 2010 in each state?
1. How much has population density changed between 2010 and 2015 in each state?
Calculate the change in percentages and map them.
1. Change the columns names in `us_states` to lowercase. (Hint: helper functions - `tolower()` and `colnames()` may help).
<!-- Mixed exercises -->
<!-- combination of use of select, mutate, group_by, summarize, etc  -->
1. Using `us_states` and `us_states_df` create a new object called `us_states_sel`.
The new object should have only two variables - `median_income_15` and `geometry`.
Change the name of the `median_income_15` column to `Income`.
1. Calculate the change in median income between 2010 and 2015 for each state.
Bonus: what was the minimum, average and maximum median income in 2015 for each region?
What is the region with the largest increase of the median income?
<!-- Raster exercises -->
1. Create a raster from scratch with nine rows and columns and a resolution of 0.5 decimal degrees (WGS84).
Fill it with random numbers.
Extract the values of the four corner cells. 
1. What is the most common class of our example raster `grain` (hint: `modal()`)?
1. Plot the histogram and the boxplot of the `data(dem, package = "RQGIS")` raster. 
1. Now attach also `data(ndvi, package = "RQGIS")`. 
Create a raster stack using `dem` and `ndvi`, and make a `pairs()` plot.