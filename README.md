# Getting and Cleaning Data - Assignment 4

## Introduction

The `run_analysis.R` script allows to download the SAMSUNG data (folder, or archive) used in assignment 4 and tidies it.

The script simply takes the SAMSUNG data as input (through variables) and outputs 2 txt files containing the tidies data.

Only sourcing the script is required to perform those operations.

By default the data folder should be located inside the `data` folder (created if missing) and the outputs are created inside a `tidy` folder.

```R
source("./run_analysis.R")
```

## How the script works

To generate a tidy dataset, the script does the following:

### Creates the source and destination folders if they don't exist

```R
data.dir <- file.path("./data") # Change to the location of the UCI HAR Dataset folder
data.tidydir <- file.path("./tidy")

# Prepare files
if (!file.exists(data.dir)) {
        dir.create(data.dir)
}

if (!file.exists(data.tidydir)) {
        dir.create(data.tidydir)
}
```

### Download the data if not present inside the source folder

```R
data.url <- "https://d396qusza40orc.cloudfront.net/getdata%2Fprojectfiles%2FUCI%20HAR%20Dataset.zip"
data.archive <- file.path(data.dir, "getdata_projectfiles_UCI_HAR_Dataset.zip")
data.filename <- file.path(data.dir, "UCI HAR Dataset")

if (!file.exists(data.archive)) {
        download.file(data.url, destfile = data)
}

if (!file.exists(data.filename)) {
        unzip(data.archive, exdir = data.dir)
}
```

### Initialize variables to be used when transforming the data

```R
# Work on dataset
library(dplyr)
library(stringr)

# test sets
data.test.file.subject <- file.path(data.filename, "test", "subject_test.txt")
data.test.file.x <- file.path(data.filename, "test", "X_test.txt")
data.test.file.y <- file.path(data.filename, "test", "y_test.txt")

# train sets
data.train.file.subject <- file.path(data.filename, "train", "subject_train.txt")
data.train.file.x <- file.path(data.filename, "train", "X_train.txt")
data.train.file.y <- file.path(data.filename, "train", "y_train.txt")

# Activity names
data.activity_names <- c('WALKING', 'WALKING_UPSTAIRS', 'WALKING_DOWNSTAIRS',
                         'SITTING', 'STANDING', 'LAYING')

# Feature ids
data.feature_ids <- c(1, 2, 3, 4, 5, 6, 41, 42, 43, 44, 45, 46, 81, 82, 83, 84, 85, 
                     86, 121, 122, 123, 124, 125, 126, 161, 162, 163, 164, 165, 166,
                     201, 202, 214, 215, 227, 228, 240, 241, 253, 254, 266, 267, 268,
                     269, 270, 271, 345, 346, 347, 348, 349, 350, 424, 425, 426, 427,
                     428, 429, 503, 504, 516, 517, 529, 530, 542, 543)

# Feature names
data.feature_names <- c("tbody_accel_mean_x", "tbody_accel_mean_y", 
                        "tbody_accel_mean_z", "tbody_accel_std_x", 
                        "tbody_accel_std_y", "tbody_accel_std_z", 
                        "tgravity_accel_mean_x", "tgravity_accel_mean_y", 
                        "tgravity_accel_mean_z", "tgravity_accel_std_x", 
                        "tgravity_accel_std_y", "tgravity_accel_std_z", 
                        "tbody_accel_jerk_mean_x", "tbody_accel_jerk_mean_y", 
                        "tbody_accel_jerk_mean_z", "tbody_accel_jerk_std_x", 
                        "tbody_accel_jerk_std_y", "tbody_accel_jerk_std_z", 
                        "tbody_gyro_mean_x", "tbody_gyro_mean_y",
                        "tbody_gyro_mean_z", "tbody_gyro_std_x",
                        "tbody_gyro_std_y", "tbody_gyro_std_z", 
                        "tbody_gyrojerk_mean_x", "tbody_gyrojerk_mean_y", 
                        "tbody_gyrojerk_mean_z", "tbody_gyrojerk_std_x", 
                        "tbody_gyrojerk_std_y", "tbody_gyrojerk_std_z", 
                        "tbody_accel_mag_mean", "tbody_accel_mag_std", 
                        "tgravity_accel_mag_mean", "tgravity_accel_mag_std", 
                        "tbody_accel_jerkmag_mean", "tbody_accel_jerkmag_std", 
                        "tbody_gyro_mag_mean", "tbody_gyro_mag_std", 
                        "tbody_gyrojerk_mag_mean", "tbody_gyrojerk_mag_std", 
                        "fbody_accel_mean_x", "fbody_accel_mean_y", 
                        "fbody_accel_mean_z", "fbody_accel_std_x", 
                        "fbody_accel_std_y", "fbody_accel_std_z", 
                        "fbody_accel_jerk_mean_x", "fbody_accel_jerk_mean_y", 
                        "fbody_accel_jerk_mean_z", "fbody_accel_jerk_std_x", 
                        "fbody_accel_jerk_std_y", "fbody_accel_jerk_std_z", 
                        "fbody_gyro_mean_x", "fbody_gyro_mean_y", 
                        "fbody_gyro_mean_z", "fbody_gyro_std_x", 
                        "fbody_gyro_std_y", "fbody_gyro_std_z", 
                        "fbody_accel_mag_mean", "fbody_accel_mag_std", 
                        "fbody_body_accel_jerk_mag_mean", 
                        "fbody_body_accel_jerk_mag_std", 
                        "fbody_body_gyro_mag_mean", "fbody_body_gyro_mag_std", 
                        "fbody_body_gyro_jerk_mag_mean", 
                        "fbody_body_gyro_jerk_mag_std")
```

### Load the files using readLines

As the values are separated by space instead of common csv separator, `readLines` has been used. The features also have some padding before the first value on each row hence the use of the `trimws` function to remove those spaces. The line is then split based on one or more spaces (`str_split` with the pattern `( +)`) and simplified to be a vector.

The datasets are converted into tables (`as_tibble`); correct columns are selected; values are converted into numeric `mutate_all` and then assigned correct column names.

A new `set` column is added to differentiate the sets if needed.

Pipline (`%>%`) is used to eliminate intermediate variables.

Columns are bound to form each sets; and then the sets rows are bound to form the final tidy dataset.

Intermediate variables are removed to free memory space and to avoid confusion.

```R
# Load test data
data.test.data.subject <- readLines(data.test.file.subject) %>%
        as_tibble %>%
        mutate(set = "TEST") %>%
        rename(subject = value)
data.test.data.x <- str_split(trimws(readLines(data.test.file.x)),
                              pattern = "( +)", simplify = TRUE) %>%
        as_tibble %>%
        select(all_of(data.feature_ids)) %>%
        mutate_all(as.numeric) %>%
        `colnames<-`(data.feature_names)
data.test.data.y <- readLines(data.test.file.y) %>%
        as_tibble %>%
        rename(activity = value) %>%
        mutate(activity = data.activity_names[as.numeric(activity)])

# Bind all test data
data.test <- bind_cols(data.test.data.subject, data.test.data.x, data.test.data.y)
# Remove variables
rm(list=c("data.test.data.subject", "data.test.data.x", "data.test.data.y"))

# Load training data
data.train.data.subject <- readLines(data.train.file.subject) %>%
        as_tibble %>%
        mutate(set = "TRAIN") %>%
        rename(subject = value)
data.train.data.x <- str_split(trimws(readLines(data.train.file.x)),
                              pattern = "( +)", simplify = TRUE) %>%
        as_tibble %>%
        select(all_of(data.feature_ids)) %>%
        mutate_all(as.numeric) %>%
        `colnames<-`(data.feature_names)
data.train.data.y <- readLines(data.train.file.y) %>%
        as_tibble %>%
        rename(activity = value) %>%
        mutate(activity = data.activity_names[as.numeric(activity)])

# Bind all training data
data.train <- bind_cols(data.train.data.subject, data.train.data.x, data.train.data.y)
# Remove variables
rm(list=c("data.train.data.subject", "data.train.data.x", "data.train.data.y"))

# Merge test and train
tidy1 <- bind_rows(data.test, data.train)

# Free intermediate
rm(list=c("data.test", "data.train"))
```

### Second tidy data is created using grouping (`group_by`) and `summarize`

The `set` column is removed from the summary using `select`.

```R
# Second tidy data
tidy2 <- tidy1 %>%
        group_by(subject, activity) %>%
        select(-set) %>%
        summarize_all(mean)

# Save files
tidy1file <- file.path(data.tidydir, "tidy1.txt")
if (file.exists(tidy1file)) {
        file.remove(tidy1file)
}
```

### Files are written into the output directory

```R
# Save files
tidy1file <- file.path(data.tidydir, "tidy1.txt")
if (file.exists(tidy1file)) {
        file.remove(tidy1file)
}
#data.table::fwrite(tidy1, "./tidy1.csv", append = FALSE)
write.table(tidy1, file = tidy1file, row.names = FALSE)

tidy2file <- file.path(data.tidydir, "tidy2.txt")
if (file.exists(tidy2file)) {
        file.remove(tidy2file)
}
#data.table::fwrite(tidy2, "./tidy2.csv", append = FALSE)
write.table(tidy2, file = tidy2file, row.names = FALSE)
```
