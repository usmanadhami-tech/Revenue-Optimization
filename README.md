# Revenue-Optimization
Identifying gap duration between offers
import pandas as pd
import pyathena
import time
import matplotlib.pyplot as plt
import numpy as np
from scipy import stats
import seaborn as sns

# Connect to Athena
conn = pyathena.connect(
    aws_access_key_id='',
    aws_secret_access_key='',
    work_group='',
    region_name='',
    endpoint_url=''
)

# Record the start time
start_time = time.time()

# Define the query
query1 = """ 
WITH CTE AS (
    SELECT 
        nm.phc_description AS abd_campaign,
        s.brand_name,
        EXTRACT(DAY FROM (abv.abd_valid_until - abv.abd_valid_from)) AS duration_of_ABD, 
        abv.*,
        LAG(abv.abd_valid_until) OVER (PARTITION BY abv.abd_ext_as_id ORDER BY abv.abd_valid_from) AS prev_abd_valid_until
    FROM 
        zoobrain_stage.abd_as_based_discounts abv
    LEFT JOIN 
        zoobrain_stage.product_history_changes nm 
        ON nm.phc_foreign_key_value = abv.abd_id
    LEFT JOIN 
        datamarts_openspace.prc_std_query_all_daily s 
        ON abv.abd_ext_as_id = s.as_id
        AND nm.phc_column IN ('ABD_DISCOUNT_VALUE','ABV_DISCOUNT_VALUE')
        AND nm.phc_foreign_key_name IN ('ABD_ID','ABV_ID')
    WHERE 
        abd_valid_from >= DATE '2023-01-01'
        AND abd_valid_until <= DATE '2024-05-31'
        AND abd_ext_u_id_closed IS NULL
        AND s.manufacturer_id = 79
        AND s.is_food = 0
)
SELECT
    abd_campaign,
    brand_name,
    duration_of_ABD,
    abd_ext_as_id,
    abd_valid_from,
    abd_valid_until,
    EXTRACT(DAY FROM (abd_valid_from - prev_abd_valid_until)) AS gap_between_promotions,
    prev_abd_valid_until
FROM CTE;
"""

# Execute the query for the original data and read the result
df1 = pd.read_sql_query(query1, conn)

# Remove NaN values from the 'gap_between_promotions' column
df1 = df1.dropna(subset=['gap_between_promotions'])

# Calculate statistical measures
mean_duration = df1['gap_between_promotions'].mean()
median_duration = df1['gap_between_promotions'].median()

# Calculate mode and handle different return types
mode_result = stats.mode(df1['gap_between_promotions'], nan_policy='omit')
if isinstance(mode_result.mode, np.ndarray) and mode_result.mode.size > 0:
    mode_duration = mode_result.mode[0]
else:
    mode_duration = mode_result.mode if mode_result.mode else 'N/A'

count_duration = df1['gap_between_promotions'].count()
std_duration = df1['gap_between_promotions'].std()
var_duration = df1['gap_between_promotions'].var()
quartiles_duration = df1['gap_between_promotions'].quantile([0.25, 0.5, 0.75])
min_duration = df1['gap_between_promotions'].min()
max_duration = df1['gap_between_promotions'].max()
range_duration = max_duration - min_duration

# Set the Seaborn color palette
sns.set_palette('dark')  # You can change 'pastel' to any other palette like 'deep', 'muted', etc.

# Create a histogram plot with Seaborn and Matplotlib
plt.figure(figsize=(12, 8))
sns.histplot(df1['gap_between_promotions'], bins=20, kde=True, edgecolor='black')

# Customize the plot
plt.xlabel('Frequency of ABD in Days', fontsize=14)
plt.ylabel('Count of Shop Articles', fontsize=14)
plt.title('Distribution of Frequency of ABD for OB Accessories since 2023', fontsize=16)
plt.grid(True)

# Add text labels for each bar
counts, bins = np.histogram(df1['gap_between_promotions'], bins=20)
for count, bin in zip(counts, bins[:-1]):
    plt.text(bin, count, str(int(count)), va='bottom', ha='left', fontsize=10, color='black')

# Add text box with statistics
textstr = (f'Mean: {mean_duration:.2f}\n'
           f'Median: {median_duration}\n'
           f'Mode: {mode_duration}\n'
           f'Count: {count_duration}\n'
           f'Standard Deviation: {std_duration:.2f}\n'
           f'Variance: {var_duration:.2f}\n'
           f'25th Percentile (Q1): {quartiles_duration[0.25]}\n'
           f'50th Percentile (Q2): {quartiles_duration[0.5]}\n'
           f'75th Percentile (Q3): {quartiles_duration[0.75]}\n'
           f'Min: {min_duration}\n'
           f'Max: {max_duration}\n'
           f'Range: {range_duration}')
props = dict(boxstyle='round', facecolor='white', alpha=0.5)
plt.gca().text(0.95, 0.95, textstr, transform=plt.gca().transAxes, fontsize=12,
               verticalalignment='top', horizontalalignment='right', bbox=props)

# Save the plot as an image
plt.savefig('frequency_of_ABD_histogram_ACC_since_2023_pastel.png', dpi=300, bbox_inches='tight')
plt.show()

# Create an Excel writer
# Print a message indicating the export is complete
print("Data exported to 'ABD_frequency_acc.xlsx'")

# Record the end time
end_time = time.time()
