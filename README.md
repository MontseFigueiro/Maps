###PLOTTING MAPS IN R 
####Shape Files - SpatialPolygonsDataFrame

**Libraries**
```r
library(ggmap)
library(ggplot2)
library(httr)
library(maps)
library(maptools)
library(sp)
library(rgdal)
library(reshape2)
install.packages("RColorBrewer")
```
*Using geocode to extract the coordinates from a geolocation*
```r
codigos <- geocode("Spain")
comunidades <- c("Andalucia","Aragon","Asturias","Baleares","Canarias","Cantabria","Castilla La Mancha","Castilla y Leon","Catalunya","Comunidad de Madrid","Comunidad Murciana","Comunidad Valenciana","Extremadura","La Rioja","Galicia","Navarra","Pais Vasco")
coordccaa <- geocode(comunidades)
map <- get_map(coordccaa,source="google",zoom=6,maptype = "terrain")
```
####Spain population by provinces in 2015
####Data Source INE
[Data source](http://www.ine.es/jaxiT3/Tabla.htm?t=2852)
We have a file with two variables (province, population), the encoding in read.xlsx I have put UTF-8 for not to be problems with accents. 

```r
library(xlsx)
datos <- read.xlsx("Poblacionprov2015.xls",sheetName = 1,encoding= "UTF-8") 
class(datos$Poblacion2015)
```
The data.frame looks like this:

 Provincia |Poblacion2015
 ----------|-------------
         02 Albacete |       394580
 03 Alicante/Alacant  |     1855047
          04 Almería   |     701211
      01 Araba/Álava    |    323648
         33 Asturias     |  1051229
            05 Ávila      |  164925

We extract the first two numbers and place them in a new column (codigo) and delete all accents. 
Later we will merge to incorporate data to SpatialPolygonDataFrame by this "codigo".
```r
datos$codigo <- substr(datos$Provincia,1,2)
datos$Provincia <- substring(datos$Provincia,4)
datos$Provincia <-  chartr('áéíóúÁÉÍÓÚñàèìòù','aeiouAEIOUnaeiou',datos$Provincia)
datos$codigo <- as.factor(datos$codigo)
class(datos$codigo)
```
Now the data looks like this:

Provincia| Poblacion2015| codigo
---------|--------------|-------
   Albacete    |    394580  |   02
 Alicante/Alacant |      1855047 1    03
          Almeria  |      701211  |   04
      Araba/Alava   |     323648   |  01
         Asturias    |   1051229   |  33
           Avila      |  164925    | 05

####Spain Map by provinces.The readShapePoly reads data from a polygon shapefile into a SpatialPolygonsDataFrame object.

[censos2011](http://www.ine.es/censos2011_datos/cen11_datos_resultados_seccen.htm)
```r
getinfo.shape(file.choose("SECC_CPV_E_20111101_01_R_INE"))
file<- readShapeSpatial("SECC_CPV_E_20111101_01_R_INE")
plot(file)
``` 
####Download Shapefile by provinces

[Shape file provinces Spain](http://www.arcgis.com/home/item.html?id=83d81d9336c745fd839465beab885ab7)
Read the shape file into a Spatialpolygondataframe:
```r
provincias <- readShapePoly("Provincias_ETRS89_30N")
provincias@data 
slotNames(provincias)
class(provincias@data$Codigo)#is factor
```
We have to include the data from our Population data.frame with a merge by the "codigo" column into the Spatialpolygondataframe.
```r
newdata <- merge(provincias@data, datos, by.x="Codigo", by.y="codigo")
provincias@data$Poblacion2015 <- newdata$Poblacion2015
provincias@data
```
####Plotting a Barplot with the Population in Spain in 2015
I want first the provinces with most population
```r
datos <- datos[order(-datos$Poblacion2015),]
datosgraf <- subset(datos,select=-codigo)#select all columns unless "codigo"
row.names(datosgraf) <- datosgraf$Provincia#change rows names
``` 
Plot and save in .png file
```r
png("PlotPoblacion2015.png",width = 900,height = 450)
par(mar =c(9, 5, 4, 1)) 
d <-  barplot((datosgraf$Poblacion2015)/1000,axes=FALSE,col=rainbow(52),ylab="Población",main="Población Año 2015",ylim=c(0,7000))
pts <- pretty(datosgraf$Poblacion2015/ 1000)
axis(2, at = pts, labels = paste(pts, "M", sep = ""))
axis(1, at=d,labels=row.names(datosgraf), las=2,cex.lab=4,cex.axis=0.8)
grid()
dev.off()
```
![PlotPoblacion2015](https://github.com/MontseFigueiro/Maps/blob/master/PlotPoblacion2015.png)

##Plot SpacialPolygonDataFrame
Provincias is a SpatialPolygonsdataframe.
```r
plot(provincias,col=provincias$Poblacion2015)
```
The spatialpolygonsdatagrame plot looks like this:

![Plotwithoutdata](https://github.com/MontseFigueiro/Maps/blob/master/Plotspatialpolygon.png)

```r
png("PlotPobl2015.png")
spplot(provincias,"Poblacion2015",main="Poblacion por Provincia 2015")
dev.off()
```
I have not indicated the colors.

![Plotwithdata](https://github.com/MontseFigueiro/Maps/blob/master/PlotPobl2015.png)

```r
png("PlotSpainPop2015.png")
spplot(provincias,zcol="Poblacion2015",col.regions=colorRampPalette(c("white","grey10"))(20),
       main=list(label="Población por Provincias 2015",cex=2),labels=provincias$Poblacion2015/1000)
dev.off()
```
![Plotwithcolorsanddata](https://github.com/MontseFigueiro/Maps/blob/master/PlotSpainPop2015.png)

####Combine shapefiles with ggmap
Get the Spain map from google, we can choose the maptype (hybrid, terrain, satellite, roadmap).The shape file show
the provinces.
```r
map <- get_map("spain",zoom=4,source="google",maptype="hybrid")
map <- ggmap(map)
shapefile <- readOGR(".","Provincias_ETRS89_30N")
shapefile <- spTransform(shapefile,CRS("+proj=longlat +  datum=WGS84"))
shapefile <- fortify(shapefile)
```
```r 
png("MapSpainshapeggmap.png")
map+geom_polygon(aes(x=long,y=lat,group=group),fill='grey',color='white',data=shapefile,alpha=0)
dev.off()
```
![Plotshapefileggmap](https://github.com/MontseFigueiro/Maps/blob/master/MapSpainshapeggmap.png)
