# circle_plot

Step 1: For a gentle overview of graph theory terms and visualization, see this article: https://medium.com/basecs/a-gentle-introduction-to-graph-theory-77969829ead8

Step 2. Initial input file setup
Download this repo. In the demo_data file, there are two sheets:

"demo" - This is a matrix of nodes (people, in this example) to be connected in the graph.
"node_attributes" - This is a list of attributes, one for each node, to characterize or describe the nodes. We will this as a lookup table and also for visualization.

The "name" column refers to the node identifier and will be used to label the nodes in the figure. The "research" column is an example but can be changed. You can also add additional attributes. Note that additional attributes here will not automatically be included in the plot - the code will need to be updated to display the new information.

There are two ways to set up the graph input: with R or Excel functions. In this example, we will primarily use R, but some steps can be replicated or replaced in Excel.

Step 3. Download R or use an IDE (e.g., Visual Studio Code). Set the current directory to the folder where you downloaded this repo.

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

Step 4. Change the file path location for demo_data.xlsx (i.e., where you downloaded the file on your computer). If you change the name of the Excel sheet "demo" or create a new sheet to use, you will need to update the "sheet_name" variable to reflect this change.

```r
filepath <- "/Users/manthony1/Library/CloudStorage/Box-Box/GitHub/circle_plot/demo_data.xlsx"
# Specify the name of the excel sheet to read (e.g., 'demo' or 'Sheet1'). The sheet name must be enclosed in quotations. In this example, the first sheet is named "demo".
sheet_name <- "demo" # update this name, if you change it in the Excel file
```

Step 5. Load the data in R.
If you receive the error message "Error: Sheet 'demo' not found", there may be unsaved changes in the Excel file. Verify that the Excel sheet name matches the name defined for the sheet_name variable, save the Excel file, then re-run the code chunk.

```r
data <- read_xlsx(filepath, sheet = sheet_name) %>%
    # Convert the first column to rownames
    column_to_rownames(var = "Name")
```
