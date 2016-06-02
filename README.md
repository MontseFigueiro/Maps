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
 Provincia |Poblacion2015
 ----------|-------------
         02 Albacete |       394580
 03 Alicante/Alacant  |     1855047
          04 Almería   |     701211
      01 Araba/Álava    |    323648
         33 Asturias     |  1051229
            05 Ávila      |  164925

datos$codigo <- substr(datos$Provincia,1,2)
datos$Provincia <- substring(datos$Provincia,4)#elimino los numeros delante del nombre de provincia
datos$Provincia <-  chartr('áéíóúÁÉÍÓÚñàèìòù','aeiouAEIOUnaeiou',datos$Provincia)#elimino los acentos
datos$codigo <- as.factor(datos$codigo)
class(datos$codigo)
```
###Mapa españa The readShapePoly reads data from a polygon shapefile into a SpatialPolygonsDataFrame object
http://www.ine.es/censos2011_datos/cen11_datos_resultados_seccen.htm
library(rgdal)
getinfo.shape(file.choose("SECC_CPV_E_20111101_01_R_INE"))
file<- readShapeSpatial("SECC_CPV_E_20111101_01_R_INE")
plot(file)

#por provincias http://www.arcgis.com/home/item.html?id=83d81d9336c745fd839465beab885ab7
provincias <- readShapePoly("Provincias_ETRS89_30N")
provincias@data # Aquí veo los nombres de provincias y codigo tiene 52 filas 5 columnas
slotNames(provincias)
class(provincias@data$Codigo)#es factor
newdata <- merge(provincias@data, datos, by.x="Codigo", by.y="codigo")
provincias@data$Poblacion2015 <- newdata$Poblacion2015
provincias@data
#Grafico 
head(datos)
library(reshape2)

datos <- datos[order(-datos$Poblacion2015),]
datosgraf <- subset(datos,select=-codigo)
row.names(datosgraf) <- datosgraf$Provincia


png("PlotPoblacion2015.png")
d <-  barplot((datosgraf$Poblacion2015)/1000,axes=FALSE,col=rainbow(52),ylab="Población",main="Población Año 2015",ylim=c(0,7000))
pts <- pretty(datosgraf$Poblacion2015/ 1000)
par(mar=c(6, 5, 4, 2) + 0.1)
axis(2, at = pts, labels = paste(pts, "M", sep = ""))
axis(1, at=d,labels=row.names(datosgraf), las=2,cex.lab=1,font=1,cex.axis=0.8)
grid()
dev.off()


##Plot SpacialPolygonDataFrame

plot(provincias,col=provincias$Poblacion2015)

png("PlotPobl2015.png")
spplot(provincias,"Poblacion2015",main="Poblacion por Provincia 2015")
dev.off()
install.packages("RColorBrewer")

library(RColorBrewer)
my.cols <- brewer.pal(6, "Blues")

png("PlotSpainPop2015")
spplot(provincias,zcol="Poblacion2015",col.regions=colorRampPalette(c("white","grey10"))(20),
       main=list(label="Población por Provincias 2015",cex=2),labels=provincias$Poblacion2015/1000)
dev.off()

##combinar shapefiles con ggmap
library(ggmap)
library(rgdal)
library(ggplot2)
library(maptools)
 
map <- get_map("spain",zoom=4,source="google",maptype="hybrid")
map <- ggmap(map)
shapefile <- readOGR(".","Provincias_ETRS89_30N")
shapefile <- spTransform(shapefile,CRS("+proj=longlat +  datum=WGS84"))
shapefile <- fortify(shapefile)
png("MapSpainshapeggmap.png")
map+geom_polygon(aes(x=long,y=lat,group=group),fill='grey',color='white',data=shapefile,alpha=0)
dev.off()
