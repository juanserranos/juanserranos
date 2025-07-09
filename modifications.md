# PVR System Code Modifications

## Issues Identified and Solutions

### 1. Variance Number Auto-increment Issue

**Problem**: The current system always gets the maximum variance number and adds 1, which doesn't maintain the increment pattern based on the first entered number.

**Solution**: Track the base variance number and maintain incremental sequence.

### 2. Variance Item Update Issue

**Problem**: The system creates redundant records instead of updating existing variance_item records.

**Solution**: Properly check for existing records using variance number and improve the update logic.

## Code Modifications

### Modification 1: Add Base Variance Number Tracking
**Location**: Class-level variables
**Tag**: `VAR_BASE_TRACKING`

```csharp
// === Add these class-level variables after the existing ones ===
private int baseVarianceNumber = 0; // Tracks the base number for auto-increment
private bool isBaseNumberSet = false; // Flag to check if base number is established
```

**Reason**: We need to track the base variance number that was first entered to maintain proper increment sequence.
**Outcome**: Enables consistent incremental numbering based on user's initial input.

### Modification 2: Update Auto-increment Logic in Save Method
**Location**: `btnSaveRecord_Click` method, Step 2 section
**Tag**: `VAR_INCREMENT_LOGIC`

Replace this section:
```csharp
// === Step 2: Generate var_number if new ===
if (isNewRecord)
{
    SqlCommand getMaxCmd = new SqlCommand("SELECT ISNULL(MAX(CAST(var_number AS INT)), 0) FROM Main_pvr", conn);
    int nextVarNumber = (int)getMaxCmd.ExecuteScalar() + 1;
    variance_number.Text = nextVarNumber.ToString();
}
```

With:
```csharp
// === Step 2: Generate var_number if new ===
if (isNewRecord)
{
    if (!isBaseNumberSet && !string.IsNullOrEmpty(variance_number.Text))
    {
        // User manually entered a number - use this as base
        if (int.TryParse(variance_number.Text, out int userNumber))
        {
            baseVarianceNumber = userNumber;
            isBaseNumberSet = true;
        }
    }
    else if (isBaseNumberSet)
    {
        // Auto-increment from base number
        SqlCommand getMaxCmd = new SqlCommand(
            "SELECT ISNULL(MAX(CAST(var_number AS INT)), @baseNumber) FROM Main_pvr WHERE CAST(var_number AS INT) >= @baseNumber", 
            conn);
        getMaxCmd.Parameters.AddWithValue("@baseNumber", baseVarianceNumber);
        int nextVarNumber = (int)getMaxCmd.ExecuteScalar() + 1;
        variance_number.Text = nextVarNumber.ToString();
    }
    else
    {
        // No base set and no user input - use traditional max + 1
        SqlCommand getMaxCmd = new SqlCommand("SELECT ISNULL(MAX(CAST(var_number AS INT)), 0) FROM Main_pvr", conn);
        int nextVarNumber = (int)getMaxCmd.ExecuteScalar() + 1;
        variance_number.Text = nextVarNumber.ToString();
        baseVarianceNumber = nextVarNumber;
        isBaseNumberSet = true;
    }
}
```

**Reason**: This logic maintains the increment pattern based on the first number entered by the user.
**Outcome**: Variance numbers will increment consistently (14000, 14001, 14002, etc.) based on the initial user input.

### Modification 3: Fix Variance Item Update Logic
**Location**: `btnSaveRecord_Click` method, Step 6 section
**Tag**: `VAR_ITEM_UPDATE_FIX`

Replace the entire Step 6 section:
```csharp
// Step 6: Save or Update Variance_item (single record version)
MessageBox.Show($"PK = '{varItemPkBox.Text}'");
bool isExisting = int.TryParse(varItemPkBox.Text, out int pk);

SqlCommand varCmd;

if (isExisting)
{
    // UPDATE
    varCmd = new SqlCommand(@"UPDATE variance_item SET 
        var_item_id = @var_item_id,
        var_description = @var_description,
        var_code = @var_code,
        disposition = @disposition,
        dispo_type = @dispo_type,
        additional_jobNumber = @additional_jobNumber,
        dispo_by = @dispo_by,
        eng_intervention = @eng_intervention,
        notes = @notes,
        var_number = @var_number
        WHERE var_item_pk = @var_item_pk", conn);

    varCmd.Parameters.AddWithValue("@var_item_pk", pk);
}
else
{
    // INSERT
    varCmd = new SqlCommand(@"INSERT INTO variance_item 
    (var_item_id, var_description, var_code, disposition, dispo_type, additional_jobNumber, dispo_by, eng_intervention, notes, var_number)
    VALUES
    (@var_item_id, @var_description, @var_code, @disposition, @dispo_type, @additional_jobNumber, @dispo_by, @eng_intervention, @notes, @var_number)", conn);
}
```

With:
```csharp
// Step 6: Save or Update Variance_item with improved logic
// First check if a variance_item record exists for this var_number
SqlCommand checkVarItemCmd = new SqlCommand("SELECT var_item_pk FROM variance_item WHERE var_number = @var_number", conn);
checkVarItemCmd.Parameters.AddWithValue("@var_number", variance_number.Text);
object existingPk = checkVarItemCmd.ExecuteScalar();

SqlCommand varCmd;
bool isUpdating = false;

if (existingPk != null)
{
    // Record exists - UPDATE
    isUpdating = true;
    varCmd = new SqlCommand(@"UPDATE variance_item SET 
        var_item_id = @var_item_id,
        var_description = @var_description,
        var_code = @var_code,
        disposition = @disposition,
        dispo_type = @dispo_type,
        additional_jobNumber = @additional_jobNumber,
        dispo_by = @dispo_by,
        eng_intervention = @eng_intervention,
        notes = @notes
        WHERE var_number = @var_number", conn);
}
else
{
    // No record exists - INSERT
    varCmd = new SqlCommand(@"INSERT INTO variance_item 
        (var_item_id, var_description, var_code, disposition, dispo_type, additional_jobNumber, dispo_by, eng_intervention, notes, var_number)
        VALUES
        (@var_item_id, @var_description, @var_code, @disposition, @dispo_type, @additional_jobNumber, @dispo_by, @eng_intervention, @notes, @var_number)", conn);
}
```

**Reason**: The original logic relied on the hidden textbox value which wasn't reliable. This approach directly checks the database for existing records based on var_number.
**Outcome**: Eliminates duplicate variance_item records and properly updates existing ones.

### Modification 4: Improve LoadVarianceItems Method
**Location**: `LoadVarianceItems` method
**Tag**: `LOAD_VAR_ITEMS_IMPROVED`

Replace the entire method:
```csharp
private void LoadVarianceItems(string varNumber)
{
    using SqlConnection conn = new SqlConnection(connectionString);
    conn.Open();

    SqlCommand cmd = new SqlCommand("SELECT * FROM variance_item WHERE var_number = @var_number", conn);
    cmd.Parameters.AddWithValue("@var_number", varNumber);

    using SqlDataReader reader = cmd.ExecuteReader();
    if (reader.Read())
    {
        MessageBox.Show("Record found for var_number = " + varNumber); // ✅ TEMP

        // Load into UI
        varitemBox.Text = reader["var_item_id"].ToString();
        jobNumberAdditionalBox.Text = reader["additional_jobNumber"].ToString();
        variance_codeBox.Text = reader["var_code"].ToString();
        dispo_typeBox.Text = reader["dispo_type"].ToString();
        disposition_byBox.Text = reader["dispo_by"].ToString();
        engineering_intervBox.Text = reader["eng_intervention"].ToString();
        varDescripBox.Text = reader["var_description"].ToString();
        DispositionBox.Text = reader["disposition"].ToString();
        NotesBox.Text = reader["notes"].ToString();

        varItemPkBox.Text = reader["var_item_pk"].ToString(); // ✅ This must run
    }
    else
    {
        MessageBox.Show("No record found for var_number = " + varNumber); // ❌ This means there's no match
    }
}
```

With:
```csharp
private void LoadVarianceItems(string varNumber)
{
    // Clear all variance item fields first
    ClearVarianceItemFields();

    using SqlConnection conn = new SqlConnection(connectionString);
    conn.Open();

    SqlCommand cmd = new SqlCommand("SELECT * FROM variance_item WHERE var_number = @var_number", conn);
    cmd.Parameters.AddWithValue("@var_number", varNumber);

    using SqlDataReader reader = cmd.ExecuteReader();
    if (reader.Read())
    {
        // Load into UI
        varitemBox.Text = reader["var_item_id"].ToString();
        jobNumberAdditionalBox.Text = reader["additional_jobNumber"].ToString();
        variance_codeBox.Text = reader["var_code"].ToString();
        dispo_typeBox.Text = reader["dispo_type"].ToString();
        disposition_byBox.Text = reader["dispo_by"].ToString();
        engineering_intervBox.Text = reader["eng_intervention"].ToString();
        varDescripBox.Text = reader["var_description"].ToString();
        DispositionBox.Text = reader["disposition"].ToString();
        NotesBox.Text = reader["notes"].ToString();

        // Ensure primary key is properly stored
        varItemPkBox.Text = reader["var_item_pk"].ToString();
    }
    // If no record found, fields remain cleared (ready for new entry)
}

// Helper method to clear variance item fields
private void ClearVarianceItemFields()
{
    varitemBox.Text = "";
    jobNumberAdditionalBox.Text = "";
    variance_codeBox.SelectedIndex = -1;
    dispo_typeBox.SelectedIndex = -1;
    disposition_byBox.SelectedIndex = -1;
    engineering_intervBox.SelectedIndex = -1;
    varDescripBox.Clear();
    DispositionBox.Text = "";
    NotesBox.Clear();
    varItemPkBox.Text = ""; // Clear the hidden primary key field
}
```

**Reason**: Removes debugging messages, adds proper field clearing, and ensures clean state management.
**Outcome**: Better user experience and proper field state management for both existing and new records.

### Modification 5: Update Final Success Message
**Location**: `btnSaveRecord_Click` method, Step 7 section
**Tag**: `SUCCESS_MESSAGE_UPDATE`

Replace:
```csharp
// === Step 7: Final Message ===
if (isNewRecord)
    MessageBox.Show("Record was saved.");
else
    MessageBox.Show($"PVR ({variance_number.Text}) was updated.");

LoadMainPVR(); // Refresh list
```

With:
```csharp
// === Step 7: Final Message ===
string varItemAction = isUpdating ? "updated" : "created";
if (isNewRecord)
    MessageBox.Show($"New PVR record ({variance_number.Text}) was saved and variance item was {varItemAction}.");
else
    MessageBox.Show($"PVR ({variance_number.Text}) was updated and variance item was {varItemAction}.");

LoadMainPVR(); // Refresh list
```

**Reason**: Provides more informative feedback about both main record and variance item actions.
**Outcome**: Better user feedback about what operations were performed.

## Summary of Changes

1. **Auto-increment Enhancement**: Variance numbers now properly increment from the user's initial input (14000 → 14001 → 14002).
2. **Update Fix**: Variance items are now properly updated instead of creating duplicates.
3. **Better State Management**: Improved loading and clearing of variance item fields.
4. **Enhanced User Feedback**: More informative success messages.

These modifications maintain the existing UI and form controls while fixing the core functional issues.