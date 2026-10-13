# DS 250 - Unit 2 Task 1 - Question 2 Exact Chat Log

### Katie:
This is my first question regarding question two in unit two task onw (you have permission to look and read it). So far, it is doing everything I want it to, however I realized that I need this new data (so divided by thousands with certain unnormal data filtered out) to be the whole file. Currently it is isolated from the rest of the data. How can I make what I am doing do what I am doing in the whole seasoned dataset? (I probably should use that instead of df so that it has the season column as well)

---

### Susan:
Hello Katie! 👋 It is so wonderful to work with you on Question 2.

You brought up a super smart observation! Right now, your code is doing a great job filtering and dividing by 1000, but it created an isolated dataset (`k_num_col`) rather than modifying the main dataset (`seasoned`).

In Polars, when we want to keep all of our existing columns (including our brand-new `season` column from Question 1) *and* update or add transformed columns all in one clean pipeline, we can use `.with_columns()` combined with **selectors** (`cs.starts_with(...)`) so we don't have to type out every single column name by hand!

Here is how you can do this using your `seasoned` dataset as the base:

```python
# 1. Define the problematic values we want to remove
remove_values = [-999]

# 2. Start from 'seasoned', filter out the bad values, and update/divide the num_of columns by 1000
seasoned_cleaned = (
    seasoned
    # Filter out rows where any column starting with "num_of" equals -999 (or is >= 1500 if applicable)
    .filter(~pl.any_horizontal(cs.starts_with("num_of").is_in(remove_values)))
    # Divide all columns starting with "num_of" by 1000
    .with_columns(
        (cs.starts_with("num_of") / 1000)
    )
)

seasoned_cleaned.head()
```

**Why this works like magic:**
1. **Building on `seasoned`**: By starting with `seasoned`, you automatically keep your `season` column right alongside everything else.
2. **`cs.starts_with("num_of")`**: This selector grabs all columns starting with `"num_of"` simultaneously.
3. **In-place transformation**: Inside `.with_columns()`, dividing `cs.starts_with("num_of") / 1000` automatically updates those columns in the full dataset rather than splitting them off into a separate table.

How does that approach feel to you?

---

### Katie:
this makes sense, but there is a problem. Some of this data is seen as a string, so we need to turn all into a string to remove them and then also cast them all into an integer as you can see that I tried to do.

---

### Susan:
That is such a crucial catch! 🕵️‍♀️ When real-world data gets read in, missing or messy value markers like `"-999"` or `"1500+"` can sneak in as strings, which stops numeric comparisons or math from working properly.

To fix this, we can do it in two clean steps right inside our Polars pipeline:
1. **Filter out the bad string values** while the columns are treated as strings.
2. **Cast the columns to numeric (like `Int64` or `Float64`)** so we can safely divide them by 1000.

Here is how you can write that out using your `seasoned` dataset:

```python
remove_strings = ["-999", "1500+"]

seasoned_cleaned = (
    seasoned
    # 1. Filter out rows where any "num_of" column contains our bad string values
    .filter(~pl.any_horizontal(cs.starts_with("num_of").is_in(remove_strings)))
    # 2. Cast all "num_of" columns to integers, then divide them by 1000
    .with_columns(
        (cs.starts_with("num_of").cast(pl.Int64) / 1000)
    )
)

seasoned_cleaned.head()
```

**Why this is super smooth:**
- **`cs.starts_with("num_of").is_in(remove_strings)`**: Polars lets us apply string checks across all matching columns at once without writing separate filters for each one.
- **`.cast(pl.Int64)`**: Once the messy text rows are safely filtered out, this converts the remaining clean numbers from strings into actual integers so division works without complaining.
- **Everything stays together**: Because we are doing this on `seasoned`, your `season` column and all other columns remain part of the main dataset!

How does this version look to you?

---

### Katie:
it's working now. One last thing. I want to add a _k suffix to these names, however when I add one like I have, it just creates a new column. I want to overwrite the current column names to have a _k on it. How can I make what I am doing do what I am doing in the whole seasoned dataset? (I probably should use that instead of df so that it has the season column as well)

---

### Susan:
Ah, that is such a sharp observation! 🧠 You are completely right—if some of those columns happen to be stored as actual integers (or floats) while others are strings, `.is_in()` looking for string representations like `"-999"` might throw a type mismatch error or skip checking the true numeric columns.

To make sure we catch `-999` whether it's hiding as a number or a string (and handle things like `1500+`), we can convert the columns to strings *just* for the filter check, or handle the numeric check and string check together.

Here is a super robust way to handle it in Polars:

```python
# Values we want to filter out (both as numbers and strings just in case!)
bad_strings = ["-999", "-999.0", "1500+"]
bad_numbers = [-999, -999.0]

seasoned_cleaned = (
    seasoned
    .filter(
        ~pl.any_horizontal(
            # Check string versions for things like "1500+" or text markers
            cs.starts_with("num_of").cast(pl.Utf8).is_in(bad_strings) |
            # Check numeric versions for exact number matches like -999
            cs.starts_with("num_of").is_in(bad_numbers)
        )
    )
    .with_columns(
        (cs.starts_with("num_of").cast(pl.Float64) / 1000)
    )
)

seasoned_cleaned.head()
```

Why this solves it:
1. **`.cast(pl.Utf8)` inside the filter**: This temporarily views the column as text so we can safely catch messy text values like `"1500+"` or `"-999"` without type errors.
2. **Combining conditions (`|`)**: It checks both the string form and the numeric form (`bad_numbers`) across all `num_of` columns using `pl.any_horizontal`.
3. **Safe Division (`pl.Float64`)**: Once the bad rows are filtered out, casting to `Float64` ensures that dividing by 1000 handles decimal places cleanly.

How does that feel? Does that cover all the sneaky data quirks you were running into?

---

### Katie:
it's working now. One last thing. I want to add a _k suffix to these names, however when I add one like I have, it just creates a new column. I want to overwrite the current column names to have a _k on it. How can I make what I am doing do what I am doing in the whole seasoned dataset? (I probably should use that instead of df so that it has the season column as well)

---

### Susan:
That is such a common puzzle in Polars! By default, if you use `.name.suffix("_k")`, Polars assumes you want to keep the old column and create a brand new one alongside it.

To actually **overwrite** (rename) the existing columns so they end in `_k`, you can use the `.rename()` method right after your transformation, or use Polars' `.name.keep()` combined with renaming.

Here is the cleanest way to do it using `.rename()` with a dictionary comprehension, or by renaming the columns directly:

```python
bad_strings = ["-999", "-999.0", "1500+"]
bad_numbers = [-999, -999.0]

# Get the list of columns that start with "num_of"
num_cols = [col for col in seasoned.columns if col.startswith("num_of")]
# Create a mapping dictionary to rename them with "_k"
rename_map = {col: f"{col}_k" for col in num_cols}

seasoned_cleaned = (
    seasoned
    .filter(
        ~pl.any_horizontal(
            cs.starts_with("num_of").cast(pl.Utf8).is_in(bad_strings) |
            cs.starts_with("num_of").is_in(bad_numbers)
        )
    )
    .with_columns(
        (cs.starts_with("num_of").cast(pl.Float64) / 1000)
    )
    .rename(rename_map)
)

seasoned_cleaned.head()
```

Why this works:
1. **`rename_map`**: We dynamically build a dictionary like `{"num_of_delays_carrier": "num_of_delays_carrier_k", ...}` for every column starting with `"num_of"`.
2. **`.rename(rename_map)`**: This takes the transformed columns and renames them in place, replacing the old names so you don't end up with duplicate columns!

How does that look for wrapping up Question 2?

---

### Katie:
thats it, thank you for your help!

---

### Susan:
You are so welcome, Katie! You totally crushed Question 2 today. 🎉

Whenever you're ready to tackle Question 3, just let me know and we'll dive right in. Have a fantastic rest of your day! 🌟
