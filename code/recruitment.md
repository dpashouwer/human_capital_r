---
title: "Recruitment"
author: "Dustin Pashouwer"
date: "Jun 30, 2018 (Replace with date uploaded to OpenSDP)"
output: 
  html_document:
    theme: simplex
    css: ../includes/styles.css
    highlight: NULL
    keep_md: true
    toc: true
    toc_depth: 3
    toc_float: true
    number_sections: false
---

# Recruitment
*Caption*

*Programmed in R*

## Getting Started



<div class="navbar navbar-default navbar-fixed-top" id="logo">
<div class="container">
<img src="../img/open_sdp_logo_red.png" style="display: block; margin: 0 auto; height: 115px;">
</div>
</div>

### Analysis
                     |
#### Loading the OpenSDP Dataset and R Packages

This guide takes advantage of several key R packages. The first chunk of code below loads the R packages (make sure to install first!), and the second chunk loads the dataset.**Feel free to add R packages to this, so long as they are common and trustworthy**



## Analyses

#### Calculate the Share of Teachers Who Are New Hires ----------------------------------------

**Purpose:** Describe the share of teachers in the agency who are new hires.

**Required Analysis File Variables:**

- `tid` 
- `school_year`        
- `t_newhire` 
- `t_novice`      

**Analysis-specific sample restrictions**

- Keep only years for which new hire information is available.

**Ask Yourself**

- How is your workforce balanced between novice and veteran teachers? Is the ratio what you expected?
- What are the major sources of novice new hires in your agency? Experienced new hires?
- How does your recruiting strategy affect the composition of your teacher workforce?

**Potential further analyses**

You can use a pie chart like this one to examine the overall distribution of various characteristics of your teacher workforce. For example, you can use a pie chart to examine categorical variables such as teacher gender, race, or tenure status, or group continuous variables such as in-district experience, total teaching experience, or teacher age into three to seven categories and then display the share of teachers in each category.


```r
# // Step 1: Load the Teacher_Year_Analysis data file.
teacher_year_analysis <- read_dta(here("data/Teacher_Year_Analysis.dta"))

# // Step 2: Restrict the analysis sample.
teacher_year_analysis %<>%
  filter(
    school_year > 2010,
    !is.na(t_newhire),
    !is.na(t_novice),
    !is.na(t_experience)
  )

# // Step 3: Review variables.
teacher_year_analysis %>% tabyl(t_newhire)
```

```
 t_newhire    n   percent
         0 1480 0.8012994
         1  367 0.1987006
```

```r
teacher_year_analysis %>% tabyl(t_novice)
```

```
 t_novice    n    percent
        0 1771 0.95885219
        1   76 0.04114781
```

```r
teacher_year_analysis %>% tabyl(t_newhire, t_novice)
```

```
 t_newhire    0  1
         0 1480  0
         1  291 76
```

```r
# // Step 4: Define a new variable which includes both novice and experienced new hires.
teacher_year_analysis %<>%
  mutate(
    pie_hire = case_when(
      t_newhire == 0 ~ "Experienced Teachers",
      t_newhire == 1 & t_novice == 0 ~ "Experienced New Hires",
      t_newhire == 1 & t_novice == 1 ~ "Novice New Hires"
    ),
    pie_hire = factor(pie_hire, levels = c("Experienced Teachers", "Experienced New Hires", "Novice New Hires"))
  )

# // Step 5: Calculate and store sample sizes for the chart footnote.

teacher_year_analysis %>%
  tabyl(pie_hire) %>%
  ggplot(aes(x = " ", y = n, fill = pie_hire)) +
  geom_bar(stat = "identity") +
  geom_text(aes(label = scales::percent(percent, accuracy = 1)), position = position_stack(vjust = 0.5)) +
  labs(
    title = "Share of Teacher Who Are New Hires",
    fill = NULL
  ) +
  theme_tntp_2018(
    grid = FALSE,
    axis_text = FALSE
  )
```

<img src="../figure/E_unnamed-chunk-1-1.png" style="display: block; margin: auto;" />

#### Examine the Share of New Hires Across School Years ----------------------------------------

**Purpose:** Examine trends in hiring over time.

**Required Analysis File Variables:**

- `tid` 
- `school_year`        
- `t_newhire` 
- `t_novice`      

**Analysis-specific sample restrictions**

- Keep only years for which new hire information is available.

**Ask Yourself**

- How have hiring trends changed over time?
- What factors might account for the trends that I see?

**Potential further analyses**

At the state level, yout may wish to exammine hiring trends by year for specific school types or geographic areas. At the district level, you can make a graph of this type in order to examine overall hiring by school or for specific groups of schools, instead of by year.


```r
# // Step 1: Load the Teacher_Year_Analysis data file.
teacher_year_analysis <- read_dta(here("data/Teacher_Year_Analysis.dta"))

# // Step 2: Restrict the analysis sample.
teacher_year_analysis %<>%
  filter(
    school_year > 2007,
    !is.na(t_newhire),
    !is.na(t_novice),
    !is.na(t_experience)
  )

# // Step 3:  Generate veteran new hire indicator.
teacher_year_analysis %<>%
  mutate(
    t_newhire_type = case_when(
      t_newhire == 1 & t_novice == 1 ~ "Novice New Hire",
      t_newhire == 1 & t_novice == 0 ~ "Experienced New Hire",
      TRUE ~ "Returned Teachers"
    ),
    t_newhire_type = t_newhire_type %>%
      factor(levels = c("Novice New Hire", "Experienced New Hire", "Returned Teachers"))
  )

# // Step 4:  Review variables to be used in the analysis.



# // Step 5:  Calculate counts and percentages table.
tabyl_school_year_t_newhire_type <- teacher_year_analysis %>% 
  count(school_year, t_newhire_type) %>% 
  group_by(school_year) %>% 
  mutate(percent = n / sum(n)) %>% 
  ungroup()

# // Step 6:  Calculate significance indicator variables by year.
tabyl_school_year_t_newhire_type_chisq <- tabyl_school_year_t_newhire_type %>% 
  select(-percent) %>% 
  spread(t_newhire_type, n) %>% 
  as_tabyl() %>% 
  chisq.test()

tabyl_school_year_t_newhire_type_sig <- tabyl_school_year_t_newhire_type_chisq$residuals %>%
  mutate_at(vars(-school_year), ~ ifelse(abs(.) >= 1.96, "*", "")) %>% 
  gather(t_newhire_type, sig, -school_year) %>% 
  mutate(school_year = as.double(school_year), 
         t_newhire_type = as.factor(t_newhire_type))

# // Step 7:  Concatenate values and significance asterisks to make value labels.
school_year_by_t_newhire_type <- left_join(tabyl_school_year_t_newhire_type, tabyl_school_year_t_newhire_type_sig) %>% 
  mutate(label = paste0(scales::percent(percent, accuracy = 1), sig))

# // Step 8:  Create a stacked bar graph using overlaid bars.
school_year_by_t_newhire_type %>% 
  ggplot(aes(x = school_year, y = percent, fill = t_newhire_type)) + 
    geom_bar(stat = "identity") + 
    geom_text(aes(label = label), position = position_fill(0.5)) + 
    theme_tntp_2018() + 
    scale_fill_tntp()
```

<img src="../figure/E_unnamed-chunk-2-1.png" style="display: block; margin: auto;" />

```r
# Q: I use the chi-square test here, which I think is a measurement away from the mean.
# They use a regression with comparison to 2008 as a base year in the OpenSDP example. Can you do this with chi-squared?
```

#### Compare the Shares of New Hires Across School Poverty Quartiles ----------------------------------------

**Purpose:** Examine the extent to which new hires are distributed unevenly across the agency according to school characteristics.

**Required Analysis File Variables:**

- `tid` 
- `school_year`        
- `t_newhire` 
- `t_novice`
- `school_poverty_quartile`

**Analysis-specific sample restrictions**

- Keep only years for which new hire information is available.

**Ask Yourself**

- How do hiring patterns differ between high and low-poverty schools?
- Are the shares of novice and veteran hires distributed equitably and strategically across school poverty quartiles?

**Potential further analyses**

You can use a version of this graph to look at how new hires are distributed across other quartiles of school characteristics. For example, you can examine new hiring by school average test score quartile, or school minority percent quartile.



#### Examine the Distribution of Teachers and Students by Race ----------------------------------------

**Purpose:** Compares the shares of all teachers, newly hired teachers, and students by race.

**Required Analysis File Variables:**

- `tid` 
- `school_year`        
- `t_newhire` 
- `t_race_ethnicity`
- `sid`
- `s_race_ethnicity`

**Analysis-specific sample restrictions**

- For the student and teacher samples, keep only records for which race information is not missing.
- For the student and teacher samples, keep only years for which teacher new hire information is available.

**Ask Yourself**

- Is the racial composition of your teacher workforce similar to the racial composition of your student body? Is there a difference in racial composition between all teachers and newly hired teachers?
- If there is a difference between teachers and students, what impact might this have on student learning?

**Potential further analyses**

You may wish to replicate this analysis for specific schools or groups of schools.


```r
# // Step 1: Load the Teacher_Year_Analysis data file.
teacher_year_analysis <- read_dta(here("data/Teacher_Year_Analysis.dta"))

# // Step 3: Restrict the teacher sample.
teacher_year_analysis %<>%
  filter(
    school_year == 2011,
    !is.na(t_race_ethnicity)
  )

# // Step 4: Review teacher variables.


# // Step 5: Get teacher sample sizes.
tabyl_t_race_ethnicity <- teacher_year_analysis %>%
  tabyl(t_race_ethnicity) %>%
  select(
    race_ethnicity = t_race_ethnicity,
    t_total = percent
  )

tabyl_new_t_race_ethnicity <- teacher_year_analysis %>%
  filter(t_newhire == 1) %>%
  tabyl(t_race_ethnicity) %>%
  select(
    race_ethnicity = t_race_ethnicity,
    t_newhire = percent
  )

# // Step 6: Load the Student_School_Year data file to get student data.
student_teacher_year_analysis <- read_dta(here("data/Student_Teacher_Year_Analysis.dta"))

# // Step 7: Make the file unique by sid and school_year.
student_teacher_year_analysis %<>%
  distinct(sid, school_year, .keep_all = TRUE)

# // Step 8: Restrict the student sample.
student_teacher_year_analysis %<>%
  filter(
    school_year == 2011,
    !is.na(s_race_ethnicity)
  )

# // Step 9: Get student sample sizes.
tabyl_s_race_ethnicity <- student_teacher_year_analysis %>%
  tabyl(s_race_ethnicity) %>%
  select(
    race_ethnicity = s_race_ethnicity,
    s_total = percent
  )

# // Step 10: Join teacher and student data tables.
race_ethnicity_tabyl <- full_join(tabyl_t_race_ethnicity, tabyl_new_t_race_ethnicity, by = "race_ethnicity") %>%
  full_join(tabyl_s_race_ethnicity, by = "race_ethnicity")

# // Step 2: Factorize the race/ethnicity categories.
race_ethnicity_tabyl <- race_ethnicity_tabyl %>%
  filter(race_ethnicity %in% c(1, 2, 3, 5)) %>%
  mutate(race_ethnicity = case_when(
    race_ethnicity == 1 ~ "Black",
    race_ethnicity == 2 ~ "Asian",
    race_ethnicity == 3 ~ "Latino",
    race_ethnicity == 5 ~ "White"
  ))

# // Step 2: Graph the results.
race_ethnicity_tabyl %>%
  gather(type, percent, -race_ethnicity) %>%
  ggplot(aes(race_ethnicity, percent, fill = type)) +
  geom_bar(stat = "identity", position = position_dodge()) +
  geom_text(aes(label = scales::percent(percent, accuracy = 1)), position = position_dodge())
```

<img src="../figure/E_unnamed-chunk-4-1.png" style="display: block; margin: auto;" />




