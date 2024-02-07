# TBIA_API_datadownload
TBIA API 資料下載程式碼範例

```R
#package
library(tidyverse)
library(jsonlite)
library(data.table)
library(progress)

#create output folder
dir.create("output")

#set api key
#apikey <- c("")

#taxon ID
splist <- ""
#note: 編號從TaiCOL搜尋
splist <- c("t0032116", "t0085944")

#set location
location <- ""
#specific polygen
location="120.018768 22.864787,120.018768 23.704895,121.002045 23.704895,121.002045 22.864787,120.018768 22.864787"

#Taiwan polygen
location="122.03063964843751 25.88687092463608,121.11877441406251 25.480936665268178,120.53649902343751 25.103476581724465,119.33898925781251 24.144717243892796,119.00939941406251 23.61228384938877,118.86657714843751 23.148410113700283,119.17419433593751 22.92598635988469,119.34997558593751 22.611950834736227,119.8663330078125 22.17516527187432,120.30578613281251 21.777833030006345,120.61340332031251 21.481664342451023,120.84411621093751 21.44076595964159,121.83288574218751 21.66556467586286,122.0086669921875 21.91040056141738,121.92077636718751 22.561232446329967,121.81091308593751 22.966454331427176,121.84387207031251 23.39063379029987,121.92077636718751 23.903886757544182,122.20642089843751 24.595051023918302,122.3272705078125 25.222801361656437,122.44812011718751 25.659321987762954,122.31628417968751 25.90663714205052,122.03063964843751 25.88687092463608"

#set eventDate
eventDate <- ""
#range
eventDate <- "2021-06-01, 2023-08-16"

#set data reques tprogress bar
total_progress <- progress_bar$new(total = length(splist), 
                                   format = "[:bar] :percent | ETA: :eta | :current/:total datasets downloaded")


#API request for loop####
for(i in 1:length(splist)){
  dir.create(sprintf("output\\%s", splist[i]))
  tryCatch({
    occ_url <- sprintf("https://tbiadata.tw/api/v1/occurrence?taxonID=%s&polygon=POLYGON%20((%s))&eventDate=%s&apikey={%s}&limit=1000&offset=0",
                       splist[i],
                       location,
                       eventDate,
                       apikey
    )
    while(TRUE) {
      tryCatch({
        occ <- fromJSON(occ_url)
        break
      }, error = function(e) {
        message("Error occurred: ", e)
        message("Retrying after 5 seconds")
        Sys.sleep(1)
      })
    }
    occ_data <- occ$data
    fwrite(occ_data, sprintf("output\\%s\\%s_p0.csv", splist[i], splist[i]))
    
    pg<-floor(occ$meta$total/1000)
    if(occ$meta$total%%1000=="0"){
      pg <- pg-1 #避免頁數剛好整除導致後面迴圈出錯
    }

    if(occ$meta$total>1000){ #Occurrence 大於1000筆
      page_progress <- progress_bar$new(total = pg, 
                                        format = "[:bar] :percent | ETA: :eta | Page :current/:total downloaded")
      for(j in 1:pg){
        offset<-1000*j
        while(TRUE) {
          tryCatch({
            occ_url <- sprintf("https://tbiadata.tw/api/v1/occurrence?taxonID=%s&polygon=POLYGON%20((%s))&eventDate=%s&apikey={%s}&limit=1000&offset=%s",
                               splist[i],
                               location,
                               eventDate,
                               apikey,
                               offset
            )
            occ <- fromJSON(occ_url)
            break
          }, error = function(e) {
            message("Error occurred: ", e)
            message("Retrying after 5 seconds")
            Sys.sleep(1)
          })
        }
        occ_data <- occ$data
        fwrite(occ_data, 
               sprintf("output\\%s\\%s_p%d.csv", splist[i], splist[i], j))
        page_progress$tick() # 更新頁面進度條
      }
    }else if(occ$meta$total>0){
      cat(i)
    }
    total_progress$tick() # 更新總進度條
    print(paste("finsh dataset ", splist[i], " download"))
  }, error=function(e){cat("ERROR :",conditionMessage(e), "\n")})
}

#讀取結果彙整成單一dataframe
file_name <- list.files("./output/", recursive=TRUE, full.names = TRUE, all.files = TRUE, include.dirs = TRUE, no..= FALSE) %>% 
  str_subset(., pattern = "csv")

file_data <-lapply(file_name, function(x)
    fread(x, sep = ",", colClasses = "character", encoding= "UTF-8"))
file_data<-data.table::rbindlist(file_data, fill = TRUE)

#save file
fwrite(file_data, "output\\TBIA_occ.csv")
```
