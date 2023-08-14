# 第 7 章 錯誤處理

### **使用例外事件而非回傳錯誤程式碼**

```java
public class DeviceController{
    ...
    public void sendShuntDown(){
        DeviceHandel handle = getHandel(DEV1);
        //Check the ststus of the device
        if(handle != DeviceHandel.INVAILD){
            //Save the device status to the record field
            retrieveDeviceRecord(handle);
            //if not suspended ,shut down
            if(record.getStatus()!= DEVICE_SUSPENDED){
                pauseDevice(handle);
                clearDeviceWorkQueue(handle);
                closeDevice(handle);
            } else{
                logger.log("Device suspended .Unable to shut down");
            }
        }else{
            logger.log("Invalid handel for :"+DEV.toString());
        }
    }
    ...
}
```

上面的程式碼問題在於，在呼叫這段程式碼之後還必須立刻檢查錯誤，如果返回了錯誤的log，呼叫者還要做出不同的邏輯處理，但因為程式本身順利執行，所以這個步驟也是常常被忽略。

再來看另一段改寫過後的程式碼，我們將自行定義了一個例外實體，並在函式發生例外時拋出，這樣事情便會變得簡單許多，原本糾纏在一起的兩個事件邏輯(裝置關閉演算法和錯誤處理)，現在被巧妙分開了。

```java
public class DeviceController{
    ...

    public void sendShutDown(){
        try{
            tryToShutDown();
        }catch(DeviceShutDownError e){
            logger.log(e);
        }
    }
    private void tryToShutDown() throws DeviceShutDownError{
        DeviceHandle handle = getHandle(DEV1);
        DeviceRecord record = retrieveDeviceRecord(handle);
        pauseDevice(handle);
        clearDeviceWorkQueue(handle);
        closeDevice(handle);
    }

    private DeviceHandle getHandle(DeviceID id){
        ...
        throw new DeviceShutDownError("Invalid handle for:" + id.toString());
        ...
    }
}
```

### **養成將try catch finally寫在程式碼開頭的習慣**

透過try catch finally我們可以包裹可能會發生問題、或是預防程式發生問題時終止應用程式運作的狀況，所以應該養成在try區塊裡放置主要的程式邏輯的習慣。

### **從呼叫者的角度定義例外類別**

當我們在應用程式裡定義例外類別時，應該關心的是，他們如何被捕獲的。

下面的程式碼中，依照每個可能會拋出的例外都進行了捕獲，雖然沒有錯，但卻包含著許多重複的程式碼，也無法進行複用。

```java
CMEPort port = new ACMEPort(12);
try{
    paro.open();
} catch(DeviceResponseException e){
    reportPortError(e);
    logger.log("Device response exception" ,e);
}catch(ATM1212UnlockedException e){
    reportPortError(e);
    logger.log("Unlock exception" ,e);
}catch(GMXError e){
    reportPortError(e);
    logger.log("Device response exception");
}finally{
    ...
}
```

但如果改成以下的寫法，我們將可能會拋出例外的程式碼邏輯獨立包裹成一個LocalPort類別，當進行呼叫時，同時就已經做好了錯誤處理。

```java
LocalPort port = new LocalPort(12);
try{
    port.open();
}catch(PortDeviceFailure e){
    reportError(e);
    logger.log(e.getMessage(),e);
}finally{
    ...
}

public void LocalPort{
    private ACMEPort innerPort;
    public LocalPort(int portNumber){
        innerPort = new ACMEport(portNumber);
    }

    public void open(){
        try{
            innerPort.open();
        }catch(DeviceResponseException e){
            reportPortError(e);
            logger.log("Device response exception" ,e);
        }catch(ATM1212UnlockedException e){
            reportPortError(e);
            logger.log("Unlock exception" ,e);
            }catch(GMXError e){
            reportPortError(e);
            logger.log("Device response exception");
        }finally{
        ...
        }
    }
}
```

### **定義程式流程**

```java
try{
    MealExpenses expenses = expenseReportDAO.getMeals(employee.getID());
    m_totle += expenses.getTotal();
}catch(MealExpensesNotFound e){
    m_total += getMealPerDiem();
}
```

上面的程式碼中，如果肉類被消費了，那肉類消費會在總消費中，反之，員工會得到一日的餐點補貼。但有沒有發現奇怪的地方，這兩個處理其實都是屬於邏輯，卻被放到了不同的區塊中。

如果改成下面的方式，則會讓整體邏輯清晰許多，我們透過修改ExpenseReportDAO，當今天沒有消費肉類的話，就讓他回傳一個MealExpense物件。

這樣的寫法稱作"特殊情況模式(Special Case Pattern)"，建立一個類別，讓他可以幫你處理特殊情況。

![Untitled](%E7%AC%AC%207%20%E7%AB%A0%20%E9%8C%AF%E8%AA%A4%E8%99%95%E7%90%86%207d94c8ada2c34a9eacaaa50c691f41be/Untitled.png)

總而言之，就是，

**不要把Exception Handling當作if else來用!!**

```java
MealExpenses expenses = expenseReportDAO.getMeals(employee.getID());
m_totle += expenses.getTotal();

public class PerDiemMealExpenses implements MealExpenses{
    public int getTotal(){
        ...
    }
}
```

### **不要回傳Null值**

如果打算在方法中返回null值，不如拋出異常，或是返回特例對象。因為當今天程式可能出現null值，我們就必須加上一層一層的null判斷if else，防止空值出現。

- Worst code
    
    ```java
    public void registerItem(Item item) {
        if (item != null) {
            ItemRegistry registry = peristentStore.getItemRegistry();
            if (registry != null) {
                Item existing = registry.getItem(item.getID());
                if (existing.getBillingPeriod().hasRetailOwner()) {
                    existing.register(item);
                }
            }
        }
    }
    ```
    
- Better code
    - 相對回傳null，我們可以回傳一個空陣列。
    
    ```java
    public List getList(){
        ...
        if(lis != null){
            return lis;
        }else {
            return Collections.emptyList();
        }
    }
    ```
    

### **不要傳遞null值**

- 在方法中返回null值是相當糟糕的做法，但將null值傳遞給其他方法就更糟糕了。
- 沒有好方法能對付由呼叫者意外傳入的null，比較恰當的做法就是禁止傳入null值。