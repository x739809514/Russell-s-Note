## Requirements

**Problem**: When updating a game, old save files may become incompatible with new data structures (e.g., adding new collectibles, skill trees, or fields).

**Goal**: Implement forward compatibility so that:
1. Old saves work with new game versions
2. New fields are properly initialized with defaults
3. Players get appropriate compensation/access to new content

## Core Strategy

### 1. Version Management

**Separate concerns**: Save structure version ≠ Game version

```csharp
public class SaveData
{
    // Save structure version (manual increment)
    public int saveVersion = 1;
    
    // Optional: game version for logging
    public string gameVersion = Application.version;
    
    // Actual data
    public PlayerData playerData;
    public List<LevelData> levels;
}
```

**Version increment rules**:
- V1: Initial release
- V2: Added collectibles system → increment to 2
- V3: Added skill tree → increment to 3
- Only increment when save structure changes, not for every game update

### 2. Migration System

**Key concept**: Don't "detect" new fields, use default values + upgrade logic

```csharp
public class SaveDataMigrator
{
    public static SaveData Migrate(SaveData data)
    {
        int originalVersion = data.saveVersion;
        
        // Sequential upgrades
        if (data.saveVersion < 2)
        {
            MigrateToV2(data);
            data.saveVersion = 2;
        }
        
        if (data.saveVersion < 3)
        {
            MigrateToV3(data);
            data.saveVersion = 3;
        }
        
        Debug.Log($"Migrated from v{originalVersion} to v{data.saveVersion}");
        return data;
    }
    
    // V1→V2: Add collectibles system
    private static void MigrateToV2(SaveData data)
    {
        foreach (var level in data.levels)
        {
            if (level.collectibles == null)
                level.collectibles = new List<string>();
            
            // Strategy for completed levels
            if (level.isCompleted)
            {
                level.hasNewContent = true; // Mark for replay
                // OR: Auto-grant collectibles
                // level.collectibles.AddRange(GetDefaults(level.id));
            }
        }
    }
    
    // V2→V3: Add skill tree
    private static void MigrateToV3(SaveData data)
    {
        if (data.playerData.unlockedSkills == null)
        {
            data.playerData.unlockedSkills = new List<string>();
            
            // Compensate with skill points
            int completed = data.levels.Count(l => l.isCompleted);
            data.playerData.skillPoints = completed * 2;
        }
    }
}
```

### 3. Save Manager

```csharp
public class SaveManager : MonoBehaviour
{
    public SaveData LoadGame()
    {
        if (!HasSave())
            return CreateNewSave();
        
        try
        {
            SaveData data = DeserializeFromFile();
            
            // Auto-upgrade if needed
            if (data.saveVersion < CURRENT_VERSION)
            {
                data = SaveDataMigrator.Migrate(data);
                SaveGame(data); // Save upgraded version
            }
            
            return data;
        }
        catch (Exception e)
        {
            Debug.LogError($"Load failed: {e.Message}");
            BackupCorruptedSave();
            return CreateNewSave();
        }
    }
    
    public void SaveGame(SaveData data)
    {
        data.saveVersion = CURRENT_VERSION;
        SerializeToFile(data);
    }
}
```

## Handling New Content for Completed Levels

### Strategy Options

1. **Mark for replay** (non-intrusive)
   ```csharp
   level.hasNewContent = true;
   // Show "NEW" badge in level select UI
   ```

2. **Auto-grant** (for critical items)
   ```csharp
   if (item.isCritical)
       data.inventory.Add(item);
   ```

3. **Compensation** (alternative rewards)
   ```csharp
   data.compensationCurrency += item.value;
   ```

4. **Unlock free-play mode**
   ```csharp
   level.canReplay = level.isCompleted;
   ```

### Recommended: Tiered Approach

```csharp
switch (newItem.importance)
{
    case Critical:
        // Auto-grant to inventory
        data.inventory.Add(item);
        break;
    
    case Major:
        // Mark level for replay
        level.hasNewContent = true;
        break;
    
    case Minor:
        // Optional flag only
        level.hasOptionalContent = true;
        break;
}
```

## Data Structure Example

```csharp
[Serializable]
public class PlayerData
{
    // V1: Base data
    public int health = 100;
    public Vector3 position;
    
    // V2: Collectibles (with defaults)
    public int totalCollectibles = 0;
    public List<string> collectedItems = new List<string>();
    
    // V3: Skill tree
    public int skillPoints = 0;
    public List<string> unlockedSkills = new List<string>();
    
    // Ensure non-null after deserialization
    public void Validate()
    {
        collectedItems ??= new List<string>();
        unlockedSkills ??= new List<string>();
    }
}
```

## Best Practices

1. ✅ **Manual version control**: Increment only on structure changes
2. ✅ **Sequential migrations**: V1→V2→V3, never skip versions
3. ✅ **Default values**: All new fields must have sensible defaults
4. ✅ **Save after upgrade**: Prevent re-running migrations
5. ✅ **Logging**: Track migration process for debugging
6. ✅ **Backup**: Keep corrupted saves for recovery
7. ✅ **Validation**: Check data integrity after load
8. ✅ **Downgrade protection**: Handle newer saves gracefully

## Common Pitfalls

❌ Using game version instead of save version  
❌ Forgetting to increment version number  
❌ Not providing defaults for new fields  
❌ Skipping migration steps  
❌ No null checks after deserialization  

## Quick Checklist

When adding new features:
- [ ] Add fields with default values
- [ ] Increment `CURRENT_SAVE_VERSION`
- [ ] Add `MigrateToVX()` method
- [ ] Decide compensation strategy for old players
- [ ] Test with old save files
- [ ] Update documentation
