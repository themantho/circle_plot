# circle_plot

### Step 1: For a gentle overview of graph theory terms and visualization,

see this article: https://medium.com/basecs/a-gentle-introduction-to-graph-theory-77969829ead8

### Step 2. Initial input file setup

Download this repo. In the demo_data file, there are two sheets:

"demo" - This is a matrix of nodes (people, in this example) to be connected in the graph.
"node_attributes" - This is a list of attributes, one for each node, to characterize or describe the nodes. We will this as a lookup table and also for visualization.

The "name" column refers to the node identifier and will be used to label the nodes in the figure. The "research" column is an example but can be changed. You can also add additional attributes. Note that additional attributes here will not automatically be included in the plot - the code will need to be updated to display the new information.

There are two ways to set up the graph input: with R or Excel functions. In this example, we will primarily use R, but some steps can be replicated or replaced in Excel.

### Step 3. Download R or use an IDE (e.g., Visual Studio Code). Set the current directory to the folder where you downloaded this repo.

Next, install and load the pacman library to assist with installing other libraries, then load the additional libraries with pacman.

```r
install.packages("pacman")
library(pacman)

# Install and load additional libraries
pacman::p_load(
    magrittr,
    tidyverse,
    data.table,
    janitor,
    stringr,
    listr,
    vroom,
    readr,
    readxl,
    igraph,
    ggraph,
    sjmisc
)
```

### Step 4. Change the file path location for demo_data.xlsx (i.e., where you downloaded the file on your computer). If you change the name of the Excel sheet "demo" or create a new sheet to use, you will need to update the "sheet_name" variable to reflect this change.

```r
filepath <- "/Users/manthony1/Library/CloudStorage/Box-Box/GitHub/circle_plot/demo_data.xlsx"
# Specify the name of the excel sheet to read (e.g., 'demo' or 'Sheet1'). The sheet name must be enclosed in quotations. In this example, the first sheet is named "demo".
sheet_name <- "demo" # update this name, if you change it in the Excel file
```

### Step 5. Load the data in R.

If you receive the error message "Error: Sheet 'demo' not found", there may be unsaved changes in the Excel file. Verify that the Excel sheet name matches the name defined for the sheet_name variable, save the Excel file, then re-run the code chunk.

```r
data <- read_xlsx(filepath, sheet = sheet_name) %>%
    # Convert the first column to rownames
    column_to_rownames(var = "Name")
```

### Step 6: Load the lookup table.

We can lookup the type of research each person does (human, animal, cross-species). The sheet called "node_attributes" should be filled in before loading the data and is assumed to be in the same Excel file as the "demo" sheet.

```r
lookup = read_xlsx(filepath, sheet = "node_attributes")
```

### Step 7: Create a graph as a shortcut to extract unique connections.

A list of connections is called an edge list in graph theory. An undirected graph means that the connections do not have directionality (i.e., a->b and b->a are the same direction. The connection is considered bidirectional, so the direction does not matter).

Note: If your graph connections have directionality, then change the mode from "undirecte" to "directed".

```r
# Keep diag = FALSE - this excludes self-connections and keeps the graph tidy and easier to interpret.
edge_list <- graph.adjacency(as.matrix(data), mode = "undirected", diag = FALSE) %>%
    igraph::as_edgelist()
```

### Step 8: Define edge attributes for visualization.

Edge attributes are characteristics of the connections between nodes. This example

```r
edge_attr <- edge_list %>%
    as_tibble() %>%
    set_colnames(c("name", "name_y")) %>%
    # Look up each person's research type in the first column, joining by the variable "name"
    left_join(lookup, by = "name") %>%
    # Rename the "name" variable with a placeholder, so that the second column can be labeled "name"
    rename(name_x = name, name = name_y) %>%
    # Look up the research type in the second column, joining by the variable "name"
    left_join(lookup, by = "name") %>%
    rename(name_y = name) %>%
    mutate(
        # Add a variable for research collaboration type (research conducted between pairs of people)
        research_collab = case_when(
            # Label human-human collaborations as "human"
            research.x == "Human" & research.y == "Human" ~ "Human",
            # Label animal-animal collaborations as "animal"
            research.x == "Animal" & research.y == "Animal" ~ "Animal",
            # Label human-animal, animal-human, or cross-species collaborations as "cross-species"
            research.x == "Human" & research.y == "Animal" ~ "Cross-species",
            research.x == "Animal" & research.y == "Human" ~ "Cross-species",
            research.x == "Cross-species" | research.y == "Cross-species" ~ "Cross-species"
        )
    )
```
