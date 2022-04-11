# Recurring Traffic Congestion Prediction Module Overview
Given a date (predStartTime), the program outputs the congestion prediction from next 24 hours to next 10 weekdays. 
* Frequency of calling the module: Once a day at midnight
* Query database for historical speed -- Start time: 2 weeks from yesterday at 12 AM. End time: 11:45 PM of yesterday 
* The size of the data queried -- Number of TMC: 13084. Size: Around 1.1G

# Input of the Module
Variables
| Variable Name        | Description           | Example  |
| ------------- |:-------------:| -----:|
| predStartTime      | The day when prediction starts | '2/2/2019' |
| TMC_list      | If we provide an empty list, the output will contain the predictions of ALL TMCs. If we provide a list of TMCs, the output will only include predictions of the specified TMCs      | Default: []. Alt: ['112P51337']    |
| loc_wd | The folder that saves the weekday models (predicting next 1-12 weekdays)      |  'models/'    |
| loc_wk | The folder that saves the weekend models (predicting next 1-12 weekends)      |  'models_weekend/'    |

File 1: raw_reading = 'data/Readings.csv'
* tmc_code: TMC ID (INRIX)
* measurement_tstamp: time of the recorded speed (UTC-6)
* speed: speed (mph) of the current time (confidence score indicates whether the speed is inferred or directly from real-time)
* reference_speed: typical free flow speed for the segment (speed limit)
![Input file example](https://user-images.githubusercontent.com/46463367/162666540-8c1032e0-8ba7-4ce0-b0fb-e31a7ff79c76.png)


File 2: clsFile = 'class_complete.csv' (Including all the TMC and their labels and coordinates)
* TMC: The same TMC ID as of tmc_code
* TMC_HERE: The TMC ID used by HERE. They are mainly the same but INRIX have more segments. INRIX usually divides one TMC defined in HERE into 2.
* lat_x: Middle of the start and end point latitude
* lon_x: Middle of the start and end point longitude
* label: The label or class of the TMC

# How the Module Works
    Step 1: Read raw dataset in csv used to create independent variables (read_raw)
        Input: location (location that saves the raw daily speeds) [string]
               tmcList, default is empty [List]
               startTime, date of next day  [example: 1/1/2020]
               endTime, date of the end of the requested time [example: 1/1/2020]
        Output: the historical speed between the start and end date [pandas data frame]
    Step 2: Preprocessing raw independent variables
    Step 3: Combine the processed independent variables (speed related and event related) together with the target dates data
        Input
            startTime: date of tomorrow [exmaple: '3/11/2019']
            historicalDT: the processed dataset [pandas data frame]
            nextday: If predicting next i day, nextday is i. For example, nextday is 1 if we want to predict tomorrow's traffic [range from 1 to 12]
            classFile (location+filename that saves the classes of the TMCs) [csv]
        Output
            input dataframe ready to put into the model [pandas data frame]
            input feature names [list of strings]
    Step 4: Prediction (pred)
        Input:
            dt_input: the output data frame from prepare_new [pandas dataframe]
            loc_weekday: the location of all the weekday models
            loc_weekend: the location of all the weekend models
            nextday: If predicting next i day, nextday is i. For example, nextday is 1 if we want to predict tomorrow's traffic [range from 1 to 12]
        

    Step 5:Running the above functions
        Input:
            predStartTime = '2/2/2019' # The day of next day
            raw_loc = 'data/' # The folder containing the historical speed csv file
            TMC_list = [] 
                # Use case 1: If we provide en empty list, the output will contain the predictions of ALL TMCs.
                # Use case 2: If we provide a list of TMCs, the output will only include predictions of the specified TMCs (an example below)
                    # TMC_list = ['112P51337'] # '112P04196' '112N04194' Can produce results of a specific TMC for testing.
            loc_wd = 'models/' # The folder that saves the weekday models (predicting next 1-12 days)
            loc_wk = 'models_weekend/' # The folder that saves the weekend models 
            raw_reading = 'INRIX data/Harris/data_raw/Readings.csv' # Historical speed
            clsFile = 'data/class_complete.csv'
        
            

# Output File
![longtermoutput](https://user-images.githubusercontent.com/46463367/162666730-2b653b41-17cd-4f47-a77d-db22ba09b010.png)

# General Information of XGBoost Models
* If predicting tomomrrow which is weekday, the models in folder 'models' will be utilzied. 
    * Input: speed from 1-5 weekdays ago, 2 weeks ago at the same time, and from 1-5 days ago 45, 30, 15 minutes ago
    * Output: tomorrow's congestion level by cluster, midnight, AM and PM (10PM-5AM, 5AM-12PM, 12PM-10PM), Mon-Thu and Fri
        
* If predicting weekends (tomorrow or within future 6 days), the models in the folder 'models_weekdns' will be utilzied.
    * Input: speed from 7 days ago at the same time, 45, 30, 15 minutes ago
    * Output: tomorrow's congestion level by cluster, AM and PM
    
* If predicting weekends (next week), the models in the folder 'models_weekdns' will be utilzied.
    * Input: speed from 14 days ago at the same time
    * Output: tomorrow's congestion level by cluster, AM and PM

* If predicting next 2-5 weekdays, the models in the folder 'models_future' whose prefix is 'model_week1*' will be utilized.
    * Input: speed from 7 days ago (5 weekdays ago) at the same time, 45, 30, 15 minutes ago
    * Output: next 5 days' congestion level by cluster, midnight, AM and PM
        
* If predict next 5-10 weekdays, the models in the folder 'models_future' whose prefic is 'model_week2*' will be utilzied.
    * Input: speed from 14 days ago (5 weekdays ago) at the same time
    * Output: next 5 days' congestion level by cluster, midnight, AM and PM
