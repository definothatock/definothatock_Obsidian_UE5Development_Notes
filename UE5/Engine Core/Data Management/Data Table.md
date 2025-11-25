In Unreal Engine (UE), a **Data Table** is a structure used to store and manage large collections of structured data, often in a tabular format. It is a powerful feature that allows you to organize and retrieve data efficiently, primarily for gameplay systems. Here's a breakdown of what a Data Table is in Unreal Engine and how it relates to C++:

---

### **What is a Data Table in Unreal Engine?**

- A **Data Table** is essentially a table (like a spreadsheet) where each row corresponds to a specific instance of data, and columns represent the properties of that row.
    
- Data Tables are often used for managing gameplay data, such as:
    
    - Character stats (health, damage, speed, etc.).
        
    - Inventory items (names, descriptions, icons, etc.).
        
    - Enemy attributes or spawn settings.
        
    - Dialogue text or localization.
        

Each row in the Data Table is represented by a data structure (or struct) in C++, and the table is typically populated using a CSV or JSON file.

---

### **How Does It Relate to C++?**

In C++, a Data Table is backed by a custom struct that defines the format of rows in the table. It is **not the same as an enum**, but it is closely tied to custom structs. Here's how it works:

1. **Struct Definition**  
    You define a USTRUCT in C++ to specify the structure of each row in the Data Table. For example:
    
    ```
    USTRUCT(BlueprintType)
    struct FCharacterStats
    {
        GENERATED_BODY()
    
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FString Name;
    
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        int32 Health;
    
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        int32 AttackPower;
    
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        float Speed;
    };
    ```
    
    This struct defines the columns of the table.
    
2. **Creating the Data Table**  
    You import a CSV or JSON file into Unreal Engine and assign it to a Data Table asset. The columns in the file correspond to the fields in the struct.
    
3. **Accessing Data in C++**  
    You can query the Data Table in C++ to retrieve specific rows of data:
    
    ```
    UDataTable* MyDataTable = LoadObject<UDataTable>(nullptr, TEXT("/Game/Data/CharacterStats.CharacterStats"));
    
    if (MyDataTable)
    {
        static const FString ContextString(TEXT("CharacterStats Context"));
        FCharacterStats* Row = MyDataTable->FindRow<FCharacterStats>(FName(TEXT("Warrior")), ContextString, true);
    
        if (Row)
        {
            UE_LOG(LogTemp, Warning, TEXT("Name: %s, Health: %d"), *Row->Name, Row->Health);
        }
    }
    ```
    
4. **Blueprint Integration**  
    Data Tables are also fully supported in Blueprints, allowing designers to query and use the data without writing code.
    

---

### **How Is It Different from Enums?**

- **Enums** are used to define a set of named values (e.g., EWeaponType::Sword, EWeaponType::Bow).
    
- **Data Tables** are used to manage and query structured data collections. While enums can be used inside a struct as a column, they are not the same thing.
    

For example:

- Enum: Define a small, fixed set of values (e.g., weapon types).
    
- Data Table: Store detailed information about each weapon (e.g., name, damage, description) using a struct.
    

---

### **Summary**

- A **Data Table** in Unreal Engine is a structured way to store large amounts of data using a USTRUCT to define the format of rows.
    
- It is **not the same as an enum**. Enums are simple, while Data Tables are for managing complex, structured data.
    
- Data Tables integrate well with both C++ and Blueprints, making them a versatile tool for game development.

[[Data Management]]