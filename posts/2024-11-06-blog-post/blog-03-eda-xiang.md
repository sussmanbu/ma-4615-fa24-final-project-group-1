# EDA


Setting up the environment and loading the data.

``` r
library(tidyverse)
library(lubridate)
library(ggplot2)
library(viridis)
library(sf)
library(viridis)
library(scales)
library(RColorBrewer) 

crash_data <- readRDS("/Users/xfu/bu-courses-repo/cas-ma-415/ma-4615-project/posts/2024-11-06-blog-post/cleaned_dataset.rds")

glimpse(crash_data)

colnames(crash_data)

summary(crash_data)

colSums(is.na(crash_data))

head(crash_data)

str(crash_data)
```

``` r
crash_data <- crash_data %>%
  mutate(Hour = hour(Crash.Date.Time))

ggplot(crash_data, aes(x = Hour)) +
  geom_histogram(binwidth = 1, fill = "steelblue", alpha = 0.7) +
  theme_minimal() +
  labs(title = "Distribution of Crashes by Hour of Day",
       x = "Hour (24-hour format)",
       y = "Number of Crashes")

distraction_fault <- crash_data %>%
  filter(Driver.Distracted.By != "UNKNOWN") %>%
  group_by(Driver.Distracted.By, Driver.At.Fault) %>%
  summarise(count = n(), .groups = 'drop') %>%
  pivot_wider(names_from = Driver.At.Fault, values_from = count, values_fill = 0)

crash_data %>%
  filter(Vehicle.Year > 1950 & Vehicle.Year < 2024) %>%  # Remove unlikely years
  mutate(Vehicle.Age = 2024 - Vehicle.Year) %>%
  ggplot(aes(x = Vehicle.Age)) +
  geom_histogram(binwidth = 1, fill = "darkred", alpha = 0.7) +
  theme_minimal() +
  labs(title = "Age Distribution of Vehicles Involved in Crashes",
       x = "Vehicle Age (Years)",
       y = "Count")

ggplot(crash_data, aes(x = Longitude, y = Latitude)) +
  geom_hex(bins = 30) +
  scale_fill_viridis_c() +
  theme_minimal() +
  labs(title = "Geographic Distribution of Crashes",
       fill = "Crash Count")

top_makes <- crash_data %>%
  group_by(Vehicle.Make) %>%
  summarise(
    crash_count = n(),
    avg_speed_limit = mean(Speed.Limit)
  ) %>%
  arrange(desc(crash_count)) %>%
  head(10)

movement_analysis <- crash_data %>%
  group_by(Vehicle.Movement) %>%
  summarise(
    crash_count = n(),
    avg_speed = mean(Speed.Limit),
    .groups = 'drop'
  ) %>%
  arrange(desc(crash_count))

license_state_analysis <- crash_data %>%
  filter(Drivers.License.State != "") %>%
  group_by(Drivers.License.State) %>%
  summarise(
    crash_count = n(),
    at_fault_count = sum(Driver.At.Fault == "Yes"),
    fault_rate = mean(Driver.At.Fault == "Yes"),
    .groups = 'drop'
  ) %>%
  arrange(desc(crash_count))
```

Image 1 shows the temporal distribution of crashes throughout the day,
revealing two distinct peak periods: a morning rush hour spike around
8-9 AM with approximately 65 crashes, and a more sustained
afternoon/evening peak between 2-6 PM reaching up to 75 crashes per
hour. The lowest crash frequencies occur during the early morning hours
(3-5 AM) with fewer than 10 crashes, demonstrating the strong
correlation between traffic volume and crash occurrence.

Image 2 presents the age distribution of vehicles involved in crashes,
showing a right-skewed distribution with a peak for vehicles between 5-7
years old (approximately 70 crashes). The frequency gradually declines
for older vehicles, with a notable drop after 15 years and very few
vehicles over 30 years old involved in crashes, likely reflecting both
the general age distribution of vehicles on the road and potentially the
retirement of older, less safe vehicles.

Image 3 shows the geographic distribution of crashes using a hexagonal
heatmap, with crash density indicated by color intensity. The
visualization reveals several high-concentration areas (shown in yellow
and green) clustered around latitude 39.1-39.2 and longitude -77.2,
suggesting potential crash hotspots that might correspond to
high-traffic intersections or challenging road configurations. The
pattern appears to follow major transportation corridors, with crash
density generally decreasing toward the periphery of the mapped area.

``` r
vehicle_year_damage <- crash_data %>%
  filter(Vehicle.Year > 1950 & Vehicle.Year < 2024) %>%
  group_by(Vehicle.Year) %>%
  summarise(
    crash_count = n(),
    severe_damage_rate = mean(Vehicle.Damage.Extent == "DISABLING"),
    .groups = 'drop'
  )

ggplot(vehicle_year_damage, aes(x = Vehicle.Year, y = severe_damage_rate)) +
  geom_point() +
  geom_smooth(method = "loess") +
  theme_minimal() +
  labs(title = "Vehicle Age vs Severe Damage Rate",
       x = "Vehicle Year",
       y = "Rate of Severe Damage")
```

The scatter plot reveals an interesting relationship between vehicle age
and severe damage rates in crashes from 1990 to 2020. There’s a
noticeable downward trend in severe damage rates from older to newer
vehicles, with vehicles from the early 1990s showing the highest rates
(around 60-65%) and a significant variability as indicated by some
points reaching above 90%. The trend line shows a steady decline until
approximately 2010, where it reaches its lowest point of about 35%
severe damage rate, suggesting improvements in vehicle safety technology
and construction over this period. However, there’s a slight uptick in
severe damage rates for the newest vehicles (2015-2020), though this
comes with wider confidence intervals as shown by the expanding grey
band, possibly due to fewer data points or other contributing factors.
The scattered points show considerable variation around the trend line,
indicating that while vehicle age is a factor in crash severity, other
variables likely play important roles in determining crash outcomes.

``` r
# spatial clustering
# distance-based clusters of crashes
library(stats)
coords <- crash_data %>%
  select(Longitude, Latitude) %>%
  as.matrix()

# hierarchical clustering
dist_matrix <- dist(coords)
clusters <- hclust(dist_matrix, method = "complete")
crash_data$cluster <- cutree(clusters, k = 5)

ggplot(crash_data, aes(x = Longitude, y = Latitude, color = factor(cluster))) +
  geom_point(alpha = 0.6) +
  theme_minimal() +
  labs(title = "Spatial Clusters of Crashes",
       color = "Cluster")
```

This scatter plot shows the spatial distribution of crashes across
different geographical coordinates, with clusters indicated by different
colors. The data reveals distinct spatial patterns with five clearly
separated clusters. The largest concentrations appear in clusters 1
(coral), 3 (green), and 4 (blue), forming a diagonal pattern from
northeast to southwest. Cluster 3, located around longitude -77.3 and
latitude 39.2, shows the highest density of crashes, while clusters 1
and 4 appear more spread out. Two smaller clusters are also visible:
cluster 2 (olive) consists of a single point in the southwest corner,
and cluster 5 (pink) shows a sparse distribution of crashes in the
western portion of the map. The clear separation between clusters
suggests that these crashes may be concentrated around specific road
features, intersections, or high-traffic areas that could benefit from
targeted safety interventions.

``` r
# route type
route_analysis <- crash_data %>%
  filter(Route.Type != "") %>%
  group_by(Route.Type) %>%
  summarise(
    crash_count = n(),
    injury_rate = mean(ACRS.Report.Type == "Injury Crash"),
    avg_speed = mean(Speed.Limit),
    most_common_collision = names(which.max(table(Collision.Type))),
    .groups = 'drop'
  )

route_analysis_long <- route_analysis %>%
  # normalize crash_count and avg_speed to 0-1 scale for better comparison
  mutate(
    crash_count_norm = (crash_count - min(crash_count)) / (max(crash_count) - min(crash_count)),
    avg_speed_norm = (avg_speed - min(avg_speed)) / (max(avg_speed) - min(avg_speed))
  ) %>%
  # convert to long format for faceting
  pivot_longer(
    cols = c(crash_count_norm, injury_rate, avg_speed_norm),
    names_to = "metric",
    values_to = "value"
  ) %>%
  mutate(
    metric = case_when(
      metric == "crash_count_norm" ~ "Crash Count (Normalized)",
      metric == "injury_rate" ~ "Injury Rate",
      metric == "avg_speed_norm" ~ "Average Speed (Normalized)"
    )
  )

ggplot(route_analysis_long, aes(x = reorder(Route.Type, value), y = value)) +
  geom_bar(stat = "identity", fill = "steelblue", alpha = 0.8) +
  geom_text(aes(label = sprintf("%.2f", value)), 
            hjust = -0.1, 
            size = 3) +
  facet_wrap(~metric, scales = "free_y", ncol = 1) +
  coord_flip() +
  theme_minimal() +
  theme(
    panel.grid.major.y = element_blank(),
    panel.grid.minor.y = element_blank(),
    axis.title.y = element_blank(),
    strip.text = element_text(face = "bold", size = 10),
    plot.title = element_text(face = "bold", size = 12),
    plot.subtitle = element_text(size = 9, color = "gray50")
  ) +
  labs(
    title = "Route Type Analysis: Metrics Comparison",
    subtitle = "Comparing crash frequency, injury rates, and average speed limits across different route types",
    y = "Value"
  )
```

We can see some comparison of road safety metrics across different route
types using normalized values. The data reveals some striking patterns:
while Interstate routes have the highest normalized average speed (1.0)
and ramps follow with relatively high speeds (0.61), Maryland State
routes dominate in terms of crash frequency with a normalized count of
1.0, followed closely by County routes at 0.86, while other route types
show significantly lower crash counts. Interestingly, when it comes to
injury rates, Other Public Roadways show the highest rate at 0.44,
followed by Maryland State routes at 0.38, suggesting that while State
routes have more crashes overall, the likelihood of injury is actually
higher on public roadways. This could indicate that while high-speed
routes like Interstates have safety measures that effectively prevent
injuries despite their speed, lower-speed but less controlled
environments might pose a higher risk for severe outcomes when crashes
do occur.

``` r
library(tidyverse)
library(lubridate)
library(scales)

# hourly summary
crash_summary <- crash_data %>%
  mutate(Hour = hour(Crash.Date.Time)) %>%
  group_by(Hour) %>%
  summarise(
    Total_Crashes = n(),
    Not_Distracted = sum(Driver.Distracted.By == "NOT DISTRACTED"),
    Distracted = sum(Driver.Distracted.By != "NOT DISTRACTED" & 
                    Driver.Distracted.By != "UNKNOWN"),
    Unknown = sum(Driver.Distracted.By == "UNKNOWN")
  ) %>%
  # convert to long format for line plotting
  pivot_longer(
    cols = c(Not_Distracted, Distracted, Unknown),
    names_to = "Distraction_Status",
    values_to = "Count"
  )

ggplot(crash_summary, aes(x = Hour, y = Count, color = Distraction_Status)) +
  geom_line(size = 1.2) +
  geom_point(size = 3) +
  geom_text(data = crash_summary %>%
              group_by(Distraction_Status) %>%
              slice_max(Count, n = 1),
            aes(label = Count),
            vjust = -0.5,
            size = 3.5) +
  scale_color_manual(
    values = c("Not_Distracted" = "#66C2A5",
              "Distracted" = "#FC8D62",
              "Unknown" = "#8DA0CB"),
    labels = c("Not Distracted", "Distracted", "Unknown Status")
  ) +
  scale_x_continuous(
    breaks = seq(0, 23, 2),
    labels = sprintf("%02d:00", seq(0, 23, 2))
  ) +
  scale_y_continuous(
    breaks = seq(0, max(crash_summary$Count), by = 10),
    expand = c(0, 2)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    legend.position = "right",
    legend.title = element_text(face = "bold"),
    legend.text = element_text(size = 10),
    axis.title = element_text(face = "bold"),
    axis.text = element_text(size = 9),
    panel.grid.minor = element_blank(),
    legend.box.background = element_rect(color = "gray80", fill = "white")
  ) +
  labs(
    title = "Hourly Distribution of Crashes by Driver Distraction Status",
    subtitle = "Showing patterns of distracted vs. non-distracted driving crashes",
    x = "Time of Day",
    y = "Number of Crashes",
    color = "Driver\nStatus"
  )

peak_hours <- crash_summary %>%
  group_by(Hour) %>%
  summarise(
    Total = sum(Count),
    .groups = 'drop'
  ) %>%
  arrange(desc(Total)) %>%
  head(5)

print("Peak Crash Hours:")
print(peak_hours)
```

This time series plot reveals a pattern in the hourly distribution of
crashes categorized by driver distraction status. The data shows
distinct peaks in distracted driving crashes, with the highest spike of
45 crashes occurring around 15:00 (3 PM), followed by significant peaks
at 8:00 AM and 12:00 PM, suggesting a strong correlation with typical
rush hour and lunch break periods. In contrast, non-distracted crashes
and those with unknown status show more moderate fluctuations, with
their highest peaks reaching only about half the magnitude of distracted
crashes (24 and 22 crashes respectively) around 15:00-16:00. The
overnight period from 22:00 to 04:00 shows consistently low crash
numbers across all categories, indicating that despite lower traffic
volumes, the proportion of distracted driving crashes remains relatively
stable during these hours.

``` r
geo_movement_analysis <- crash_data %>%
  # filter out any invalid coordinates and empty movement patterns
  filter(!is.na(Latitude), !is.na(Longitude), 
         Vehicle.Movement != "", 
         Vehicle.Movement != "UNKNOWN") %>%
  mutate(
    Movement_Category = case_when(
      Vehicle.Movement %in% c("MOVING CONSTANT SPEED", "ACCELERATING", "SLOWING OR STOPPING") ~ "Normal Traffic Flow",
      Vehicle.Movement %in% c("PARKING", "PARKED", "BACKING") ~ "Parking Related",
      Vehicle.Movement %in% c("MAKING LEFT TURN", "MAKING RIGHT TURN", "U-TURN") ~ "Turning Movements",
      TRUE ~ "Other Movements"
    )
  )

ggplot(geo_movement_analysis) +
  geom_hex(aes(x = Longitude, y = Latitude, fill = ..count..), 
           bins = 30) +
  scale_fill_viridis(option = "plasma", name = "Crash Count") +
  facet_wrap(~Movement_Category) +
  theme_minimal() +
  theme(
    legend.position = "right",
    plot.title = element_text(face = "bold"),
    plot.subtitle = element_text(color = "grey30"),
    strip.text = element_text(face = "bold")
  ) +
  labs(
    title = "Geographic Distribution of Crashes by Movement Type",
    subtitle = "Spatial patterns of different vehicle movements leading to crashes",
    x = "Longitude",
    y = "Latitude"
  )

hotspot_analysis <- geo_movement_analysis %>%
  group_by(Movement_Category) %>%
  summarise(
    crash_count = n(),
    avg_speed_limit = mean(Speed.Limit),
    injury_crashes = sum(ACRS.Report.Type == "Injury Crash"),
    injury_rate = injury_crashes / n(),
    .groups = 'drop'
  ) %>%
  arrange(desc(crash_count))

print(hotspot_analysis)

movement_time_analysis <- geo_movement_analysis %>%
  mutate(Hour = hour(Crash.Date.Time)) %>%
  group_by(Movement_Category, Hour) %>%
  summarise(
    crash_count = n(),
    .groups = 'drop'
  )

ggplot(movement_time_analysis, 
       aes(x = Hour, y = crash_count, color = Movement_Category)) +
  geom_line(size = 1) +
  geom_point(size = 2) +
  scale_x_continuous(breaks = 0:23) +
  scale_color_viridis_d() +
  theme_minimal() +
  theme(
    legend.position = "bottom",
    axis.text.x = element_text(angle = 45)
  ) +
  labs(
    title = "Crash Patterns Throughout the Day by Movement Type",
    x = "Hour of Day",
    y = "Number of Crashes",
    color = "Movement Type"
  )
```

The temporal distribution of crashes by movement type shows a clear
dominance of normal traffic flow incidents throughout the day, with two
prominent peaks: one during the morning rush hour around 9:00 (39
crashes) and an even higher peak during the evening rush hour at 18:00
(45 crashes). Other movement types (turning movements, parking-related,
and other movements) maintain relatively consistent but lower crash
frequencies, typically ranging between 5-15 crashes per hour, with
subtle increases during daylight hours. The pattern suggests that while
regular traffic flow poses the highest crash risk, especially during
peak commuting hours, the other movement types contribute a persistent
but lower baseline of crash incidents, with turning movements showing
slightly higher numbers than parking-related incidents during business
hours, particularly between 13:00-17:00.

``` r
vehicle_analysis <- crash_data %>%
  # filter out invalid years and create vehicle age
  filter(Vehicle.Year > 1950 & Vehicle.Year < 2024) %>%
  mutate(
    Vehicle.Age = 2024 - Vehicle.Year,
    Age.Category = case_when(
      Vehicle.Age <= 5 ~ "New (0-5 years)",
      Vehicle.Age <= 10 ~ "Recent (6-10 years)",
      Vehicle.Age <= 15 ~ "Mature (11-15 years)",
      TRUE ~ "Older (15+ years)"
    ),
    Age.Category = factor(Age.Category, 
                         levels = c("New (0-5 years)", 
                                  "Recent (6-10 years)", 
                                  "Mature (11-15 years)", 
                                  "Older (15+ years)"))
  )

ggplot(vehicle_analysis, aes(x = Age.Category, fill = Vehicle.Damage.Extent)) +
  geom_bar(position = "fill") +
  scale_fill_viridis_d() +
  theme_minimal() +
  theme(
    axis.text.x = element_text(angle = 45, hjust = 1),
    legend.position = "right",
    plot.title = element_text(face = "bold")
  ) +
  labs(
    title = "Vehicle Age Category vs. Damage Extent",
    subtitle = "Distribution of damage severity across different vehicle age groups",
    x = "Vehicle Age Category",
    y = "Proportion of Crashes",
    fill = "Damage Extent"
  )

top_manufacturers <- vehicle_analysis %>%
  group_by(Vehicle.Make) %>%
  summarise(
    total_crashes = n(),
    severe_crashes = sum(Vehicle.Damage.Extent %in% c("DISABLING", "SEVERE")),
    severe_crash_rate = severe_crashes / total_crashes,
    avg_vehicle_age = mean(Vehicle.Age),
    fault_rate = mean(Driver.At.Fault == "Yes", na.rm = TRUE),
    .groups = 'drop'
  ) %>%
  filter(total_crashes >= 20) %>%  # Filter for manufacturers with significant data
  arrange(desc(total_crashes))

ggplot(head(top_manufacturers, 10), 
       aes(x = reorder(Vehicle.Make, total_crashes), 
           y = total_crashes)) +
  geom_col(aes(fill = severe_crash_rate)) +
  geom_text(aes(label = sprintf("%.1f%%", severe_crash_rate * 100)),
            hjust = -0.1,
            size = 3) +
  coord_flip() +
  scale_fill_gradient(low = "lightblue", high = "darkred") +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold"),
    legend.position = "right"
  ) +
  labs(
    title = "Top 10 Vehicle Makes Involved in Crashes",
    subtitle = "Showing total crashes and severe crash rate percentage",
    x = "Vehicle Make",
    y = "Total Number of Crashes",
    fill = "Severe Crash Rate"
  )

print("Summary of Top Manufacturer Patterns")
manufacturer_patterns <- top_manufacturers %>%
  select(Vehicle.Make, total_crashes, severe_crash_rate, avg_vehicle_age, fault_rate) %>%
  mutate(
    severe_crash_rate = percent(severe_crash_rate, accuracy = 0.1),
    fault_rate = percent(fault_rate, accuracy = 0.1),
    avg_vehicle_age = round(avg_vehicle_age, 1)
  ) %>%
  arrange(desc(total_crashes))

print(manufacturer_patterns)
```

While Toyota and Honda lead in total crash numbers with around 150
crashes each, they maintain relatively moderate severe crash rates of
41% and 40.7% respectively. Interestingly, some manufacturers with fewer
total crashes show higher severity rates - KIA stands out with the
highest severe crash rate at 48.3%, followed by Nissan at 46.9%, despite
having significantly fewer total crashes. Chevrolet and Dodge, with the
lowest crash counts among the top 10, also show the lowest severe crash
rates at 25.9% and 24.0% respectively.

From “Vehicle Age Category vs. Damage Extent”, we can see a clear
relationship between vehicle age and damage severity in crashes. The
proportion of disabling and destroyed vehicles notably increases with
vehicle age, being most pronounced in the Older (15+ years) category.
While superficial damage remains relatively consistent across age groups
at around 25-30%, functional damage decreases with vehicle age,
particularly in the oldest category. This suggests that older vehicles
are more susceptible to severe damage in crashes, possibly due to their
aging structural integrity and lack of modern safety features.

``` r
vehicle_analysis <- crash_data %>%
  filter(Vehicle.Year > 1950 & Vehicle.Year < 2024) %>%
  mutate(
    Vehicle.Age = 2024 - Vehicle.Year,
    Age.Category = case_when(
      Vehicle.Age <= 5 ~ "New (0-5 years)",
      Vehicle.Age <= 10 ~ "Recent (6-10 years)",
      Vehicle.Age <= 15 ~ "Mature (11-15 years)",
      TRUE ~ "Older (15+ years)"
    ),
    Age.Category = factor(Age.Category, 
                         levels = c("New (0-5 years)", 
                                  "Recent (6-10 years)", 
                                  "Mature (11-15 years)", 
                                  "Older (15+ years)"))
  )

ggplot(vehicle_analysis, 
       aes(x = Age.Category, 
           fill = Vehicle.Damage.Extent)) +
  geom_bar(position = "dodge") +
  scale_fill_brewer(palette = "Set2") +
  theme_minimal() +
  theme(
    axis.text.x = element_text(angle = 45, hjust = 1),
    legend.position = "right",
    plot.title = element_text(face = "bold"),
    panel.grid.minor = element_blank()
  ) +
  labs(
    title = "Vehicle Age Category vs. Damage Extent",
    subtitle = "Number of crashes by damage type for each vehicle age group",
    x = "Vehicle Age Category",
    y = "Number of Crashes",
    fill = "Damage Extent"
  ) +
  geom_text(
    aes(label = after_stat(count)),
    stat = "count",
    position = position_dodge(width = 0.9),
    vjust = -0.5,
    size = 3
  )
```

This bar chart reveals the absolute numbers of crashes by damage type
across vehicle age categories, providing a more detailed view of crash
severity patterns. Recent vehicles (6-10 years) experience the highest
number of disabling crashes at 116 incidents, followed by new vehicles
(0-5 years) with 93 disabling crashes. Functional damage shows a
declining trend with vehicle age, from 84 cases in recent vehicles to 40
cases in older vehicles (15+ years). Superficial damage follows a
similar decreasing pattern, with the highest numbers in newer vehicles
(67 cases) declining to 41 cases in older vehicles. Interestingly, while
destroyed vehicles remain relatively low across all categories, there’s
a slight increase in the older vehicle category with 17 cases,
suggesting that while older vehicles may have fewer crashes overall,
they’re more likely to be completely destroyed when they do crash.

``` r
crash_timing <- crash_data %>%
  mutate(
    Hour = hour(Crash.Date.Time),
    Time_Block = case_when(
      Hour >= 5 & Hour < 9 ~ "Morning Peak (5-8)",
      Hour >= 9 & Hour < 16 ~ "Mid-Day (9-15)",
      Hour >= 16 & Hour < 19 ~ "Evening Peak (16-18)",
      Hour >= 19 & Hour < 22 ~ "Evening (19-21)",
      TRUE ~ "Night (22-4)"
    ),
    Time_Block = factor(Time_Block, 
                       levels = c("Morning Peak (5-8)", 
                                "Mid-Day (9-15)",
                                "Evening Peak (16-18)", 
                                "Evening (19-21)",
                                "Night (22-4)")),
    Has_Injury = ACRS.Report.Type == "Injury Crash",
    Light = factor(Light, levels = c("DAYLIGHT", 
                                   "DARK LIGHTS ON",
                                   "DARK NO LIGHTS",
                                   "DAWN",
                                   "DUSK",
                                   "DARK -- UNKNOWN LIGHTING",
                                   "UNKNOWN",
                                   "OTHER",
                                   "N/A"))
  )

ggplot(crash_timing, aes(x = Hour)) +
  geom_line(stat = "count", aes(color = Light), size = 1) +
  scale_x_continuous(breaks = seq(0, 23, 2)) +
  scale_color_brewer(palette = "Set2") +
  scale_y_continuous(expand = c(0, 0)) +
  theme_minimal() +
  theme(
    axis.text.x = element_text(angle = 0),
    legend.position = "right",
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    panel.grid.minor = element_blank(),
    panel.grid.major.x = element_blank(),
    legend.title = element_text(face = "bold"),
    axis.title = element_text(face = "bold")
  ) +
  labs(
    title = "Crash Distribution Throughout the Day",
    subtitle = "Showing crash frequency and lighting conditions for each hour",
    x = "Hour of Day (24-hour format)",
    y = "Number of Crashes",
    color = "Light\nCondition"
  )
```

The line graph shows the temporal distribution of crashes across
different lighting conditions throughout a 24-hour period, revealing
some interesting patterns in crash frequency. Daylight crashes show two
prominent peaks: one during the morning rush hour around 8:00
(approximately 62 crashes) and a higher peak in the early afternoon
around 14:00 (approximately 75 crashes), followed by a sharp decline as
daylight fades. As expected, “Dark Lights On” crashes become prevalent
during evening hours, showing a moderate peak around 19:00-20:00 (about
28 crashes), while maintaining relatively consistent numbers throughout
the nighttime hours. Other lighting conditions, including dawn, dusk,
and dark with no lights, show notably lower crash frequencies but
exhibit small spikes during their respective natural occurrence
periods - dawn crashes peak around 6:00-7:00, and dusk-related incidents
show a minor increase around 17:00-18:00, corresponding to typical
twilight hours.

``` r
crash_summary <- crash_data %>%
  mutate(Hour = hour(Crash.Date.Time)) %>%
  group_by(Hour) %>%
  summarise(
    Total_Crashes = n(),
    Daylight_Crashes = sum(Light == "DAYLIGHT"),
    Dark_Lights_On = sum(Light == "DARK LIGHTS ON"),
    Dark_No_Lights = sum(Light == "DARK NO LIGHTS")
  ) %>%
  pivot_longer(
    cols = c(Daylight_Crashes, Dark_Lights_On, Dark_No_Lights),
    names_to = "Condition",
    values_to = "Count"
  ) %>%
  mutate(
    Condition = case_when(
      Condition == "Daylight_Crashes" ~ "Daylight",
      Condition == "Dark_Lights_On" ~ "Dark (Lights On)",
      Condition == "Dark_No_Lights" ~ "Dark (No Lights)"
    )
  )

ggplot(crash_summary, aes(x = Hour, y = Count, color = Condition)) +
  geom_line(size = 1.2) +
  geom_point(size = 3) +
  scale_color_manual(
    values = c("Daylight" = "#4E79A7",
              "Dark (Lights On)" = "#F28E2B",
              "Dark (No Lights)" = "#59A14F")
  ) +
  scale_x_continuous(
    breaks = seq(0, 23, 2),
    labels = sprintf("%02d:00", seq(0, 23, 2))
  ) +
  scale_y_continuous(
    breaks = seq(0, 80, 10),
    expand = c(0, 2)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    legend.position = "right",
    legend.title = element_text(face = "bold"),
    legend.text = element_text(size = 10),
    axis.title = element_text(face = "bold"),
    axis.text = element_text(size = 9),
    panel.grid.minor = element_blank(),
    legend.box.background = element_rect(color = "gray80", fill = "white")
  ) +
  labs(
    title = "Hourly Distribution of Crashes by Light Condition",
    subtitle = "Showing the three main lighting conditions throughout the day",
    x = "Time of Day",
    y = "Number of Crashes",
    color = "Light Condition"
  )
```

The hourly distribution of crashes by light condition reveals distinct
patterns that align with natural daylight cycles and peak traffic hours.
Daylight crashes show two prominent peaks: a morning rush hour spike
around 8:00 with approximately 62 crashes, and an even higher afternoon
peak at 14:00 (2 PM) with about 75 crashes. As daylight fades in the
evening hours after 16:00 (4 PM), there’s a clear transition where “Dark
(Lights On)” crashes become predominant, reaching their peak of around
28 crashes during the evening commute hours. Crashes in “Dark (No
Lights)” conditions remain consistently low throughout the 24-hour
period, never exceeding 5 crashes per hour, suggesting that while these
conditions are potentially hazardous, they occur less frequently or
drivers may be more cautious in unlit conditions.

``` r
driver_analysis <- crash_data %>%
  filter(
    Driver.Distracted.By != "UNKNOWN",
    Driver.Distracted.By != "N/A",
    Vehicle.Damage.Extent != "UNKNOWN",
    Vehicle.Damage.Extent != "N/A",
    Driver.At.Fault != "UNKNOWN"
  ) %>%
  mutate(
    Distraction_Type = case_when(
      Driver.Distracted.By == "NOT DISTRACTED" ~ "Not Distracted",
      str_detect(Driver.Distracted.By, "CELL|PHONE|TEXT") ~ "Phone Usage",
      str_detect(Driver.Distracted.By, "EATING|DRINKING") ~ "Eating/Drinking",
      TRUE ~ "Other Distraction"
    ),
    Damage_Level = case_when(
      Vehicle.Damage.Extent %in% c("DESTROYED", "DISABLING") ~ "Severe",
      Vehicle.Damage.Extent == "FUNCTIONAL" ~ "Moderate",
      Vehicle.Damage.Extent %in% c("SUPERFICIAL", "NO DAMAGE") ~ "Minor",
      TRUE ~ "Other"
    ),
    At_Fault = Driver.At.Fault == "Yes"
  )

distraction_summary <- driver_analysis %>%
  group_by(Distraction_Type, Damage_Level) %>%
  summarise(
    Total_Cases = n(),
    Fault_Rate = mean(At_Fault),
    .groups = 'drop'
  ) %>%
  mutate(
    Distraction_Type = factor(Distraction_Type, 
                            levels = c("Not Distracted", "Phone Usage", 
                                     "Eating/Drinking", "Other Distraction")),
    Damage_Level = factor(Damage_Level, 
                         levels = c("Minor", "Moderate", "Severe"))
  )

ggplot(distraction_summary, 
       aes(x = Distraction_Type, y = Total_Cases, fill = Damage_Level)) +
  geom_col(position = "dodge", width = 0.7) +
  geom_text(aes(label = sprintf("%.0f%%", Fault_Rate * 100)),
            position = position_dodge(width = 0.7),
            vjust = -0.5,
            size = 3.5,
            fontface = "bold") +
  scale_fill_manual(
    values = c("Minor" = "#66C2A5",
              "Moderate" = "#FC8D62",
              "Severe" = "#8DA0CB")
  ) +
  scale_y_continuous(
    expand = expansion(mult = c(0, 0.1)),
    breaks = seq(0, max(distraction_summary$Total_Cases), by = 50)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    axis.title = element_text(face = "bold"),
    axis.text.x = element_text(angle = 45, hjust = 1),
    legend.title = element_text(face = "bold"),
    panel.grid.minor = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white")
  ) +
  labs(
    title = "Driver Distraction and Crash Severity Patterns",
    subtitle = "Numbers above bars show percentage of at-fault cases",
    x = "Type of Distraction",
    y = "Number of Crashes",
    fill = "Damage\nSeverity"
  )

distraction_stats <- driver_analysis %>%
  group_by(Distraction_Type) %>%
  summarise(
    Total_Crashes = n(),
    Fault_Rate = mean(At_Fault),
    Severe_Damage_Rate = mean(Damage_Level == "Severe"),
    .groups = 'drop'
  ) %>%
  arrange(desc(Total_Crashes))

print(distraction_stats)
```

This bar chart reveals patterns in crash severity across different types
of driver distraction, with percentages indicating at-fault cases for
each category. Non-distracted driving shows a relatively balanced
distribution of crash severities, but with notably lower at-fault rates
(21-33%) compared to distracted driving scenarios. The most striking
finding is that all distracted driving categories - phone usage,
eating/drinking, and other distractions - show extremely high at-fault
rates of 87-100%. Of particular concern is the “Other Distraction”
category, which not only shows high numbers of crashes across all
severity levels but also maintains extremely high at-fault rates
(94-96%). Phone usage and eating/drinking, while showing fewer total
incidents, result in 100% at-fault severe crashes, highlighting the
serious dangers of these specific distraction types.

``` r
geo_analysis <- crash_data %>%
  mutate(
    Hour = hour(Crash.Date.Time),
    Time_Category = case_when(
      Hour >= 5 & Hour < 12 ~ "Morning (5-11)",
      Hour >= 12 & Hour < 17 ~ "Afternoon (12-16)",
      Hour >= 17 & Hour < 22 ~ "Evening (17-21)",
      TRUE ~ "Night (22-4)"
    ),
    Is_Injury = ACRS.Report.Type == "Injury Crash"
  ) %>%
  filter(!is.na(Latitude), !is.na(Longitude))

ggplot(geo_analysis, aes(x = Longitude, y = Latitude)) +
  facet_wrap(~Time_Category, ncol = 2) +
  geom_hex(bins = 20, aes(fill = ..count..)) +
  scale_fill_viridis(
    option = "plasma",
    name = "Number\nof Crashes",
    labels = scales::number_format()
  ) +
  coord_cartesian(
    xlim = c(min(geo_analysis$Longitude) + 0.05, 
             max(geo_analysis$Longitude) - 0.05),
    ylim = c(min(geo_analysis$Latitude) + 0.05, 
             max(geo_analysis$Latitude) - 0.05)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    strip.text = element_text(face = "bold", size = 11),
    legend.title = element_text(face = "bold"),
    axis.title = element_text(face = "bold"),
    panel.grid.minor = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white")
  ) +
  labs(
    title = "Geographic Distribution of Crashes by Time of Day",
    subtitle = "Hexagonal heatmap showing crash concentration patterns",
    x = "Longitude",
    y = "Latitude"
  )

area_summary <- geo_analysis %>%
  mutate(
    Area = case_when(
      Longitude > median(Longitude) & Latitude > median(Latitude) ~ "Northeast",
      Longitude <= median(Longitude) & Latitude > median(Latitude) ~ "Northwest",
      Longitude > median(Longitude) & Latitude <= median(Latitude) ~ "Southeast",
      TRUE ~ "Southwest"
    )
  ) %>%
  group_by(Area, Time_Category) %>%
  summarise(
    Total_Crashes = n(),
    Injury_Rate = mean(Is_Injury),
    .groups = 'drop'
  )

ggplot(area_summary, 
       aes(x = Area, y = Total_Crashes, fill = Time_Category)) +
  geom_col(position = "dodge", width = 0.7) +
  geom_text(aes(label = sprintf("%.1f%%", Injury_Rate * 100)),
            position = position_dodge(width = 0.7),
            vjust = -0.5,
            size = 3.5) +
  scale_fill_brewer(palette = "Set2") +
  theme_minimal() +
  theme(
    axis.text.x = element_text(angle = 45, hjust = 1),
    plot.title = element_text(face = "bold"),
    legend.position = "right"
  ) +
  labs(
    title = "Crash Distribution by Area and Time",
    subtitle = "Numbers show injury rates for each category",
    x = "Area",
    y = "Number of Crashes",
    fill = "Time of Day"
  )
```

From this, we try to show the crash patterns across different geographic
areas and times of day, with injury rates displayed for each category.
The Northwest area shows the highest overall crash numbers and notably
high injury rates across all time periods, with evening crashes having a
particularly high injury rate of 41.8%. The Southeast region experiences
its highest crash volumes in the afternoon with a 24.5% injury rate, but
shows a concerning spike in injury rates during night hours at 29.7%.
Throughout all areas, night-time crashes (22-4) consistently show lower
total numbers but varying injury rates, from 10% in the Southwest to
29.7% in the Southeast, suggesting that different areas face distinct
night-time driving challenges. The Southwest area, while having fewer
total crashes, maintains relatively high injury rates during afternoon
(32.5%) and evening (37%) periods, indicating a need for targeted safety
measures during these times.

``` r
movement_analysis <- crash_data %>%
  filter(
    Vehicle.Movement != "",
    Vehicle.Movement != "UNKNOWN",
    Weather != "",
    Weather != "UNKNOWN",
    Collision.Type != "",
    Collision.Type != "UNKNOWN"
  ) %>%
  mutate(
    Movement_Type = case_when(
      Vehicle.Movement %in% c("MAKING LEFT TURN", "MAKING RIGHT TURN", "U-TURN") ~ "Turning",
      Vehicle.Movement %in% c("MOVING CONSTANT SPEED", "ACCELERATING") ~ "Moving Forward",
      Vehicle.Movement %in% c("SLOWING OR STOPPING", "STOPPED IN TRAFFIC") ~ "Slowing/Stopped",
      Vehicle.Movement %in% c("PARKING", "PARKED", "BACKING") ~ "Parking Related",
      TRUE ~ "Other"
    ),
    Weather_Type = case_when(
      Weather == "CLEAR" ~ "Clear",
      Weather %in% c("CLOUDY", "FOGGY") ~ "Cloudy/Foggy",
      Weather %in% c("RAIN", "SLEET", "SNOW") ~ "Precipitation",
      TRUE ~ "Other"
    )
  )

movement_summary <- movement_analysis %>%
  group_by(Movement_Type, Weather_Type) %>%
  summarise(
    Crash_Count = n(),
    Most_Common_Collision = names(which.max(table(Collision.Type))),
    Collision_Percentage = max(table(Collision.Type)) / n(),
    .groups = 'drop'
  ) %>%
  group_by(Weather_Type) %>%
  mutate(
    Weather_Total = sum(Crash_Count),
    Percentage = Crash_Count / Weather_Total
  ) %>%
  ungroup()

ggplot(movement_summary, 
       aes(x = Weather_Type, y = Crash_Count, fill = Movement_Type)) +
  geom_col(position = "fill", width = 0.7) +
  geom_text(aes(label = sprintf("%.1f%%", Percentage * 100)),
            position = position_fill(vjust = 0.5),
            color = "white",
            size = 3.5,
            fontface = "bold") +
  scale_fill_brewer(palette = "Set2") +
  scale_y_continuous(labels = scales::percent) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    axis.title = element_text(face = "bold"),
    legend.title = element_text(face = "bold"),
    panel.grid.minor = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white")
  ) +
  labs(
    title = "Vehicle Movement Patterns Under Different Weather Conditions",
    subtitle = "Showing the distribution of vehicle movements and their relative frequencies",
    x = "Weather Condition",
    y = "Proportion of Crashes",
    fill = "Movement\nType"
  )

collision_summary <- movement_analysis %>%
  group_by(Movement_Type, Collision.Type) %>%
  summarise(
    Count = n(),
    .groups = 'drop'
  ) %>%
  group_by(Movement_Type) %>%
  slice_max(Count, n = 2) %>%
  mutate(Percentage = Count / sum(Count))

ggplot(collision_summary,
       aes(y = reorder(Movement_Type, Count), 
           x = Count, 
           fill = Collision.Type)) +
  geom_col(position = "dodge") +
  geom_text(aes(label = sprintf("%.0f%%", Percentage * 100)),
            position = position_dodge(width = 0.9),
            hjust = -0.2,
            size = 3.5) +
  scale_fill_brewer(palette = "Set3") +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold"),
    axis.title = element_text(face = "bold"),
    legend.title = element_text(face = "bold")
  ) +
  labs(
    title = "Most Common Collision Types by Vehicle Movement",
    subtitle = "Numbers show percentage within each movement type",
    x = "Number of Crashes",
    y = "Movement Type",
    fill = "Collision Type"
  )
```

Image 1 reveals that moving forward is the predominant vehicle movement
pattern across all weather conditions, but its proportion notably
increases during precipitation (60%) compared to clear conditions
(45.6%). During cloudy/foggy conditions, there’s a marked increase in
slowing/stopped vehicles (20.9%) compared to clear weather (10.4%),
suggesting drivers are more cautious in reduced visibility. Turning
movements remain relatively consistent across weather conditions but
show a slight increase during precipitation (20%) compared to other
conditions.

Image 2 shows distinct collision patterns for each movement type. Most
notably, slowing/stopped vehicles are overwhelmingly involved in
rear-end collisions (88%), highlighting the risks of sudden stops. For
turning movements, straight movement angle collisions dominate (63%),
followed by head-on left turn accidents (37%). Moving forward vehicles
show a more balanced distribution between straight movement angle
collisions (59%) and same direction rear-end crashes (41%), while
parking-related incidents are predominantly classified as “other” types
(88%), suggesting unique accident dynamics in parking scenarios.

``` r
speed_analysis <- crash_data %>%
  mutate(
    Hour = hour(Crash.Date.Time),
    Time_Block = case_when(
      Hour >= 5 & Hour < 10 ~ "Morning Rush (5-9)",
      Hour >= 10 & Hour < 16 ~ "Mid-Day (10-15)",
      Hour >= 16 & Hour < 20 ~ "Evening Rush (16-19)",
      Hour >= 20 | Hour < 5 ~ "Night (20-4)"
    ),
    Speed_Category = case_when(
      Speed.Limit <= 25 ~ "≤25 mph",
      Speed.Limit <= 35 ~ "26-35 mph",
      Speed.Limit <= 45 ~ "36-45 mph",
      TRUE ~ ">45 mph"
    ),
    Speed_Category = factor(Speed_Category, 
                          levels = c("≤25 mph", "26-35 mph", 
                                   "36-45 mph", ">45 mph")),
    Has_Injury = ACRS.Report.Type == "Injury Crash"
  ) %>%
  filter(Speed.Limit > 0)

proportions_data <- speed_analysis %>%
  group_by(Time_Block, Speed_Category, Has_Injury) %>%
  summarise(count = n(), .groups = 'drop') %>%
  group_by(Time_Block, Speed_Category) %>%
  mutate(
    total = sum(count),
    proportion = count/total,
    label = sprintf("%.1f%%", proportion * 100)
  )

ggplot(proportions_data, 
       aes(x = Speed_Category, y = proportion, fill = Has_Injury)) +
  facet_wrap(~Time_Block, ncol = 2) +
  geom_col(position = "stack") +
  geom_text(aes(label = label),
            position = position_stack(vjust = 0.5),
            size = 3.5,
            color = "white") +
  scale_fill_manual(
    values = c("FALSE" = "#66C2A5", "TRUE" = "#FC8D62"),
    labels = c("No Injury", "Injury"),
    name = "Crash\nOutcome"
  ) +
  scale_y_continuous(labels = scales::percent) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(color = "gray30", size = 11),
    strip.text = element_text(face = "bold", size = 11),
    axis.title = element_text(face = "bold"),
    axis.text.x = element_text(angle = 45, hjust = 1),
    legend.title = element_text(face = "bold"),
    panel.grid.minor = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white")
  ) +
  labs(
    title = "Injury Rates by Speed Limit and Time of Day",
    subtitle = "Showing the proportion of injury crashes across different conditions",
    x = "Speed Limit Category",
    y = "Proportion of Crashes"
  )

speed_summary <- speed_analysis %>%
  group_by(Speed_Category) %>%
  summarise(
    Total_Category_Crashes = n(),
    Avg_Injury_Rate = mean(Has_Injury)
  ) %>%
  arrange(desc(Total_Category_Crashes))

ggplot(speed_summary,
       aes(x = Speed_Category, y = Avg_Injury_Rate)) +
  geom_col(aes(fill = Avg_Injury_Rate), width = 0.7) +
  geom_text(aes(label = sprintf("n=%d", Total_Category_Crashes)),
            vjust = -0.5,
            size = 3.5) +
  geom_text(aes(label = sprintf("%.1f%%", Avg_Injury_Rate * 100)),
            vjust = 1.5,
            color = "white",
            size = 3.5,
            fontface = "bold") +
  scale_fill_viridis(option = "plasma", guide = "none") +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold"),
    axis.title = element_text(face = "bold"),
    axis.text.x = element_text(angle = 45, hjust = 1)
  ) +
  labs(
    title = "Overall Injury Rates by Speed Limit",
    subtitle = "Numbers above bars show total crash count",
    x = "Speed Limit Category",
    y = "Average Injury Rate"
  )

print("Summary Statistics by Speed Category:")
print(speed_summary)
```

Image 1 shows a somewhat complex relationship between injury rates,
speed limits, and time of day. During evening rush hour (16-19), roads
with 36-45 mph limits show the highest injury rate at 54.7%, while
mid-day (10-15) demonstrates lower injury rates across all speed
categories, with ≤25 mph zones having the lowest at 17.9%.
Interestingly, high-speed zones (\>45 mph) show varying injury patterns,
from 0% injuries during evening rush to 44.4% during night hours (20-4),
suggesting that timing significantly influences crash severity in these
zones.

Image 2 provides a broader perspective on overall injury rates by speed
limit, revealing that the 36-45 mph zones have both the highest injury
rate (45.3%) and a substantial number of crashes (n=192). Despite
expectations, the highest speed category (\>45 mph) shows a relatively
low injury rate of 21.6%, though this may be influenced by its smaller
sample size (n=51). The 26-35 mph zones represent the highest crash
frequency (n=460) with a moderate injury rate of 37.2%, while the lowest
speed zones maintain the lowest injury rate at 20.2% across a
significant number of crashes (n=262).

``` r
collision_analysis <- crash_data %>%
  filter(
    Traffic.Control != "UNKNOWN",
    Traffic.Control != "N/A",
    Weather != "UNKNOWN",
    Collision.Type != "UNKNOWN",
    Collision.Type != "OTHER"
  ) %>%
  mutate(
    Control_Type = case_when(
      str_detect(Traffic.Control, "SIGNAL") ~ "Traffic Signal",
      str_detect(Traffic.Control, "STOP") ~ "Stop Sign",
      str_detect(Traffic.Control, "YIELD") ~ "Yield Sign",
      Traffic.Control == "NO CONTROLS" ~ "No Controls",
      TRUE ~ "Other Controls"
    ),
    Weather_Condition = case_when(
      Weather == "CLEAR" ~ "Clear",
      Weather == "CLOUDY" ~ "Cloudy",
      Weather %in% c("RAIN", "SNOW", "SLEET") ~ "Precipitation",
      TRUE ~ "Other"
    ),
    Collision_Category = case_when(
      str_detect(Collision.Type, "REAR-END") ~ "Rear End",
      str_detect(Collision.Type, "ANGLE") ~ "Angle",
      str_detect(Collision.Type, "SIDESWIPE") ~ "Sideswipe",
      str_detect(Collision.Type, "HEAD ON") ~ "Head On",
      TRUE ~ "Other"
    )
  )

control_summary <- collision_analysis %>%
  group_by(Control_Type, Weather_Condition) %>%
  summarise(
    Total_Crashes = n(),
    Injury_Rate = mean(ACRS.Report.Type == "Injury Crash"),
    .groups = 'drop'
  ) %>%
  group_by(Weather_Condition) %>%
  mutate(
    Proportion = Total_Crashes / sum(Total_Crashes)
  )

plot1 <- ggplot(control_summary, 
       aes(x = Weather_Condition, y = Proportion, fill = Control_Type)) +
  geom_col(position = position_dodge(width = 0.9), width = 0.8) +
  geom_text(aes(label = sprintf("%.1f%%", Proportion * 100)),
            position = position_dodge(width = 0.9),
            vjust = -0.5,
            size = 3.5,
            fontface = "bold") +
  geom_text(aes(label = sprintf("IR: %.1f%%", Injury_Rate * 100)),
            position = position_dodge(width = 0.9),
            vjust = 1.5,
            size = 3,
            color = "darkgray") +
  scale_fill_manual(values = c(
    "No Controls" = "#2E86C1",
    "Other Controls" = "#F1C40F",
    "Stop Sign" = "#E67E22",
    "Traffic Signal" = "#27AE60",
    "Yield Sign" = "#8E44AD"
  )) +
  scale_y_continuous(
    labels = scales::percent,
    limits = c(0, 0.8),
    breaks = seq(0, 0.8, 0.1)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 16),
    plot.subtitle = element_text(color = "gray30", size = 12),
    axis.title = element_text(face = "bold", size = 12),
    axis.text = element_text(size = 10),
    legend.title = element_text(face = "bold", size = 12),
    legend.text = element_text(size = 10),
    panel.grid.minor = element_blank(),
    panel.grid.major.x = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white", linewidth = 0.5),
    plot.margin = margin(t = 20, r = 20, b = 20, l = 20)
  ) +
  labs(
    title = "Traffic Control Effectiveness by Weather Condition",
    subtitle = "Showing crash distribution (%) and injury rates (IR)",
    x = "Weather Condition",
    y = "Percentage of Crashes",
    fill = "Traffic Control\nType"
  )

collision_type_summary <- collision_analysis %>%
  group_by(Control_Type, Collision_Category) %>%
  summarise(
    Count = n(),
    Injury_Count = sum(ACRS.Report.Type == "Injury Crash"),
    Injury_Rate = Injury_Count / Count,
    .groups = 'drop'
  ) %>%
  group_by(Control_Type) %>%
  mutate(Proportion = Count / sum(Count))

plot2 <- ggplot(collision_type_summary, 
       aes(x = Control_Type, y = Count, fill = Collision_Category)) +
  geom_col(position = position_dodge(width = 0.8), width = 0.7) +
  geom_text(aes(label = sprintf("%.1f%%", Injury_Rate * 100)),
            position = position_dodge(width = 0.8),
            vjust = -0.5,
            size = 3.5,
            fontface = "bold") +
  scale_fill_manual(values = c(
    "Angle" = "#3498DB",
    "Head On" = "#E74C3C",
    "Rear End" = "#2ECC71",
    "Sideswipe" = "#F1C40F",
    "Other" = "#95A5A6"
  )) +
  scale_y_continuous(
    breaks = seq(0, 200, 25),
    expand = expansion(mult = c(0, 0.15))
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 16),
    plot.subtitle = element_text(color = "gray30", size = 12),
    axis.title = element_text(face = "bold", size = 12),
    axis.text = element_text(size = 10),
    axis.text.x = element_text(angle = 45, hjust = 1, face = "bold"),
    legend.title = element_text(face = "bold", size = 12),
    legend.text = element_text(size = 10),
    panel.grid.minor = element_blank(),
    panel.grid.major.x = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white", linewidth = 0.5),
    plot.margin = margin(t = 20, r = 20, b = 40, l = 20)
  ) +
  labs(
    title = "Collision Types and Injury Rates by Traffic Control",
    subtitle = "Numbers above bars show injury rates (%)",
    x = "Traffic Control Type",
    y = "Number of Crashes",
    fill = "Type of\nCollision"
  )

print("Summary by Traffic Control Type:")
print(control_summary %>% 
      group_by(Control_Type) %>%
      summarise(
        Total_Crashes = sum(Total_Crashes),
        Avg_Injury_Rate = mean(Injury_Rate),
        .groups = 'drop'
      ) %>%
      arrange(desc(Total_Crashes)))

print(plot1)
print(plot2)
```

Image 1 reveals that traffic signals are the predominant control type
across all weather conditions, but their effectiveness varies
significantly. During precipitation, traffic signals account for 66.7%
of crashes with a 50% injury rate, while no-control areas show a
concerning 100% injury rate despite fewer crashes (33.3%). In clear
conditions, traffic signals handle 48.3% of crashes with a 35.8% injury
rate, while no-control areas see 38.7% of crashes with a 32% injury
rate, suggesting better performance in good weather.

Image 2 demonstrates that traffic signals experience the highest volume
of crashes but show varying injury rates by collision type - angle
collisions have a 48.9% injury rate while head-on collisions show 46.8%.
Stop signs, while having fewer crashes, show a high injury rate for
angle collisions (53.4%). Areas with no controls show relatively
balanced injury rates across collision types (35-39%), except for
sideswipes at 21.7%. Notably, yield signs have limited data but show a
50% injury rate, suggesting they may warrant closer monitoring for
safety effectiveness.

``` r
road_analysis <- crash_data %>%
  filter(
    Route.Type != "",
    !is.na(Speed.Limit),
    Speed.Limit > 0,
    ACRS.Report.Type != "UNKNOWN"
  ) %>%
  mutate(
    Road_Category = case_when(
      str_detect(Route.Type, "Interstate") ~ "Interstate",
      str_detect(Route.Type, "State|Maryland") ~ "State Road",
      str_detect(Route.Type, "County") ~ "County Road",
      str_detect(Route.Type, "Municipal") ~ "Municipal Road",
      TRUE ~ "Other"
    ),
    Speed_Category = case_when(
      Speed.Limit <= 25 ~ "Low (≤25)",
      Speed.Limit <= 35 ~ "Medium (26-35)",
      Speed.Limit <= 45 ~ "High (36-45)",
      TRUE ~ "Very High (>45)"
    ),
    Speed_Category = factor(Speed_Category, 
                          levels = c("Low (≤25)", "Medium (26-35)", 
                                   "High (36-45)", "Very High (>45)")),
    Severity = case_when(
      ACRS.Report.Type == "Property Damage Crash" ~ "Property Damage",
      ACRS.Report.Type == "Injury Crash" ~ "Injury",
      TRUE ~ "Other"
    )
  )

road_summary <- road_analysis %>%
  group_by(Road_Category, Speed_Category) %>%
  summarise(
    Total_Crashes = n(),
    Injury_Count = sum(Severity == "Injury"),
    Injury_Rate = Injury_Count / Total_Crashes,
    Avg_Speed = mean(Speed.Limit),
    .groups = 'drop'
  )

plot1 <- ggplot(road_summary, 
       aes(x = Speed_Category, y = Total_Crashes, fill = Road_Category)) +
  geom_col(position = "dodge", width = 0.8) +
  geom_text(aes(label = Total_Crashes),
            position = position_dodge(width = 0.8),
            vjust = -0.5,
            size = 3.5,
            fontface = "bold") +
  geom_text(aes(label = sprintf("IR: %.1f%%", Injury_Rate * 100)),
            position = position_dodge(width = 0.8),
            vjust = 1.5,
            size = 3,
            color = "darkgray") +
  scale_fill_manual(values = c(
    "Interstate" = "#2980B9",
    "State Road" = "#27AE60",
    "County Road" = "#F1C40F",
    "Municipal Road" = "#E67E22",
    "Other" = "#95A5A6"
  )) +
  scale_y_continuous(
    expand = expansion(mult = c(0, 0.2)),
    breaks = seq(0, max(road_summary$Total_Crashes), by = 25)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 16),
    plot.subtitle = element_text(color = "gray30", size = 12),
    axis.title = element_text(face = "bold", size = 12),
    axis.text = element_text(size = 10),
    axis.text.x = element_text(angle = 45, hjust = 1),
    legend.title = element_text(face = "bold", size = 12),
    legend.text = element_text(size = 10),
    panel.grid.minor = element_blank(),
    panel.grid.major.x = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white", linewidth = 0.5),
    plot.margin = margin(t = 20, r = 20, b = 40, l = 20)
  ) +
  labs(
    title = "Crash Distribution by Road Type and Speed Limit",
    subtitle = "Showing crash counts and injury rates (IR)",
    x = "Speed Limit Category",
    y = "Number of Crashes",
    fill = "Road Type"
  )

time_road_analysis <- road_analysis %>%
  mutate(
    Hour = hour(Crash.Date.Time),
    Time_Category = case_when(
      Hour >= 5 & Hour < 10 ~ "Morning Rush (5-9)",
      Hour >= 10 & Hour < 16 ~ "Mid-Day (10-15)",
      Hour >= 16 & Hour < 20 ~ "Evening Rush (16-19)",
      TRUE ~ "Night (20-4)"
    )
  ) %>%
  group_by(Road_Category, Time_Category) %>%
  summarise(
    Crashes = n(),
    Injury_Rate = mean(Severity == "Injury"),
    .groups = 'drop'
  )

plot2 <- ggplot(time_road_analysis, 
       aes(x = Road_Category, y = Crashes, fill = Time_Category)) +
  geom_col(position = "dodge", width = 0.8) +
  geom_text(aes(label = Crashes),
            position = position_dodge(width = 0.8),
            vjust = -0.5,
            size = 3.5,
            fontface = "bold") +
  geom_text(aes(label = sprintf("%.1f%%", Injury_Rate * 100)),
            position = position_dodge(width = 0.8),
            vjust = 1.5,
            size = 3,
            color = "darkgray") +
  scale_fill_manual(values = c(
    "Morning Rush (5-9)" = "#3498DB",
    "Mid-Day (10-15)" = "#2ECC71",
    "Evening Rush (16-19)" = "#F1C40F",
    "Night (20-4)" = "#95A5A6"
  )) +
  scale_y_continuous(
    expand = expansion(mult = c(0, 0.2)),
    breaks = seq(0, max(time_road_analysis$Crashes), by = 25)
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", size = 16),
    plot.subtitle = element_text(color = "gray30", size = 12),
    axis.title = element_text(face = "bold", size = 12),
    axis.text = element_text(size = 10),
    axis.text.x = element_text(angle = 45, hjust = 1),
    legend.title = element_text(face = "bold", size = 12),
    legend.text = element_text(size = 10),
    panel.grid.minor = element_blank(),
    panel.grid.major.x = element_blank(),
    legend.position = "right",
    legend.box.background = element_rect(color = "gray80", fill = "white", linewidth = 0.5),
    plot.margin = margin(t = 20, r = 20, b = 40, l = 20)
  ) +
  labs(
    title = "Crash Distribution by Road Type and Time of Day",
    subtitle = "Showing crash counts and injury rates (%)",
    x = "Road Type",
    y = "Number of Crashes",
    fill = "Time Period"
  )

print("Summary by Road Type:")
print(road_analysis %>% 
      group_by(Road_Category) %>%
      summarise(
        Total_Crashes = n(),
        Avg_Injury_Rate = mean(Severity == "Injury"),
        Avg_Speed_Limit = mean(Speed.Limit),
        .groups = 'drop'
      ) %>%
      arrange(desc(Total_Crashes)))

print(plot1)
print(plot2)
```

Image 1 shows that medium-speed roads (26-35 mph) experience the highest
number of crashes, with State Roads leading at 241 crashes and a 36.1%
injury rate, followed by County Roads with 179 crashes and a 39.1%
injury rate. High-speed roads (36-45 mph) show elevated injury rates,
particularly on State Roads (43.5%) and County Roads (49.1%), despite
lower crash volumes. Interestingly, very high-speed roads (\>45 mph)
show relatively few crashes but maintain significant injury rates, with
State Roads showing a 30.8% injury rate across 26 incidents.

Image 2 reveals distinct temporal patterns across road types, with State
Roads experiencing the highest crash volumes during mid-day (153
crashes, 35.3% injury rate) and evening rush (113 crashes, 38.9% injury
rate). County Roads show a similar pattern but with lower volumes,
peaking during mid-day (124 crashes, 29% injury rate). Municipal Roads
and Interstates have considerably fewer crashes but show varying injury
rates across time periods, with Municipal Roads experiencing higher
injury rates during morning rush hours (21.4%). Night-time crashes show
generally lower volumes across all road types but maintain concerning
injury rates, particularly on State Roads (29.7% across 74 crashes).