# circle_plot

## Step 1: A gentle overview of graph theory terms and visualization

See this article for conceptual background and terminology before starting: https://medium.com/basecs/a-gentle-introduction-to-graph-theory-77969829ead8

## Step 2. Initial setup

Download this repo by clicking on the green "Code" button, then "Download ZIP". Unzip the file. In the demo_data file, there are two sheets:

"demo" - This is a matrix of nodes (people, in this example) to be connected in the graph and is the main data that will be visualized. Rows and columns are researcher names, and values of "1" indicate that a research collaboration exists between researchers.

"node_attributes" - A list of node attributes or characteristics, one for each node. This will be used as a lookup table and for visualization. The "name" column refers to the node identifier and will be used to label the nodes in the figure. The "research" column is an example but can be changed. You can also add additional attributes. Note that additional attributes here will not automatically be included in the plot - the code will need to be updated to display the new information.

There are two ways to set up the graph input: with R or Excel functions. In this example, we will primarily use R, but some steps can be replicated or replaced in Excel.

## Step 3. Set up R or use an IDE (e.g., Visual Studio Code)

Set the working directory in R to the folder where you downloaded this repo with the command setwd(/path/to/project), where /path/to/project is replaced with your folder path. This will tell R where to look for data.

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

Change the file path location for demo_data.xlsx. If you change the name of the sheet "demo" or create a new sheet, you will need to update the "sheet_name" variable to reflect this change.

```r
# Change this path to the location of your copy of the file
filepath <- "/Users/manthony1/Library/CloudStorage/Box-Box/GitHub/circle_plot/demo_data.xlsx"
# Specify the name of the excel sheet to read (e.g., 'demo' or 'Sheet1'). The sheet name must be enclosed in quotations. In this example, the first sheet is named "demo".
sheet_name <- "demo"
```

## Step 5. Load the data in R

```r
data <- read_xlsx(filepath, sheet = sheet_name) %>%
    # Convert the first column to rownames
    column_to_rownames(var = "Name")
```

If you receive the error message "Error: Sheet 'demo' not found", there may be unsaved changes in the Excel file. Verify that the Excel sheet name matches the name defined for the sheet_name variable, save the Excel file, then re-run the code chunk.

## Step 6: Set up the lookup table

We can look up the type of research each person does (human, animal, cross-species). The sheet called "node_attributes" is assumed to be in the same Excel file as the "demo" sheet. If you change the data in node_attributes, you will need to rerun this code chunk for the changes to be reflected in R.

```r
lookup = read_xlsx(filepath, sheet = "node_attributes")
```

## Step 7: Create an edge list

A list of connections is called an edge list in graph theory. An undirected graph means that the connections do not have directionality (i.e., a->b and b->a are same connection and considered bidirectional, so the direction does not matter). This step will extract unique connections to plot.

Note: If your graph connections have directionality, then change the mode from "undirected" to "directed".

```r
# Keep diag = FALSE - this excludes self-connections and keeps the graph tidy and easier to interpret.
edge_list <- graph.adjacency(as.matrix(data), mode = "undirected", diag = FALSE) %>%
    igraph::as_edgelist()
```

## Step 8: Define edge attributes for visualization

Edge attributes are characteristics of the connections between nodes. This example plots research collaboration type. You can replace research collaboration with a different variable in the last section of this code chunk. Make sure also change the variable name and

```r
edge_attr <- edge_list %>%
    as_tibble() %>%
    # The lookup key is called "name", so we rename the first column as "name" and the second column with a placeholder "name_y"
    set_colnames(c("name", "name_y")) %>%
    # Look up each person's research type in the first column, joining by the variable "name"
    left_join(lookup, by = "name") %>%
    # Rename the "name" variable with a placeholder, so that the second column can be labeled "name" to use as the lookup key
    rename(name_x = name, name = name_y) %>%
    # Look up the research type in the second column
    left_join(lookup, by = "name") %>%
    rename(name_y = name) %>%
    # Use mutate to add new columns based on existing ones or format existing columns. case_when() is an R equivalent of the SQL "searched" ⁠CASE WHEN⁠ statement.
    mutate(
        # Add research collaboration type (research conducted between pairs of people)
        research_collab = case_when(
            # Label human-human collaborations as "human"
            research.x == "Human" & research.y == "Human" ~ "Human",
            # Label animal-animal collaborations as "animal"
            research.x == "Animal" & research.y == "Animal" ~ "Animal",
            # Label human-animal and animal-human collaborations as "cross-species"
            research.x == "Human" & research.y == "Animal" ~ "Cross-species",
            research.x == "Animal" & research.y == "Human" ~ "Cross-species",
             # This line finds where research.x = Cross-species OR where research.y = Cross-species and labels the row value as "Cross-species" in the research_collab column
            research.x == "Cross-species" | research.y == "Cross-species" ~ "Cross-species"
        )
    )
```

## Step 9: Create a graph from the edge list

```r
graph <- edge_list %>%
    igraph::graph_from_edgelist(directed = FALSE) %>%
    # Add individual research as a node attribute from the lookup table (this pulls data from the node_attributes Excel sheet)
    set_vertex_attr("research", value = lookup$research) %>%
    # Add research collaboration as an edge attribute
    set_edge_attr("research_collab", value = edge_attr$research_collab)
```

If you receive the error message "object of type 'closure' is not subsettable", re-run the code chunks that define the lookup, edge_attr, and node_attr variables. If any of these code chunks were not run, the below code will not be able to find the variables.

## Define plot parameters for visualization

```r
params <- list(
    edge_width = 0.6, # width of the connections (lower = thinner; min limit: 0)
    edge_alpha = 0.75, # transparency of the edges (lower = more transparent; min-max limits: [0, 1])
    node_hjust = -0.4, # amount of horizontal offset between the nodes and node labels. A more negative value will increase the amount of leading white space before the labels.
    node_shape = 21, # shape of the nodes (21 = circle)
    node_size = 3.25, # size of the nodes (higher = larger radius)
    font_family = "Arial", # you can use most standard fonts
    edge_legend_name = "Research Collaboration", # legend name for the edges
    legend_title_size = 14, # font size of the legend title
    legend_text_size = 13, # font size of the legend text
    edge_color = c("#fbbc4d", "#94d196", "#7dbeff") # define hex color codes for the edge attribute used in geom_edge_arc color aesthetic. NOTE: R sorts these alphabetically. If the alphabetically ordered values for the edge attribute are Animal, Cross-species, Human, then the order of the hex codes should be [animal color], [cross-species color], [human color]). If the colors are not assigned to the correct edges, the order of the hex codes likely needs to be adjusted.
)
```

### Plot the graph!

```r
ggraph(graph, layout = "linear", circular = TRUE) +
    # Add edges
    geom_edge_arc(
        aes(colour = research_collab), # Here the edge attribute "research_collab" is used to color code the connections
        alpha = params$edge_alpha,
        strength = params$edge_width
    ) +
    # Add nodes
    geom_node_point(
        aes(fill = research), # The node attribute research is used to color code the fill color of the nodes
        shape = params$node_shape,
        size = params$node_size
    ) +
    # Add node labels without a border around the text. To include a border around the text, comment out "geom_node_text(" and uncomment "geom_node_label("
    geom_node_text(
        # geom_node_label(
        aes(
            label = name,
            angle = node_angle(x, y)
        ),
        hjust = params$node_hjust, repel = FALSE
    ) +
    # Add color to the edges
    scale_edge_color_manual(
        values = params$edge_color,
        name = params$edge_legend_name) +
    # Add a theme to look pretty
    theme_graph() +
    # Define the graph coordinates
    coord_fixed(
        xlim = c(-1.4, 1.4), ylim = c(-1.4, 1.4)
    ) +
    guides(
        # Format the color fill of the individual research legend
        fill = guide_legend("Individual Research")
    ) +
    # Global formatting
    theme(
        legend.title = element_text(family = params$font_family, size = params$legend_title_size), # Legend title
        legend.text = element_text(family = params$font_family, size = params$legend_text_size) # Legend text
    )
```

![Alt text](/Users/manthony1/Library/CloudStorage/Box-Box/GitHub/circle_plot/graph_figure.png?raw=true "Circular plot of individual and collaborative research")
