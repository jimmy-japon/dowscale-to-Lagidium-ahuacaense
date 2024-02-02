##################################################################################################
##########                             Occupancy downscaling                              #########
###################################################################################################
#### scrip para estimar el AOO de una especie especialista, mediante la reduccion de  escala ######
#### haciendo uso de la relacion que existe entre la curva ocupacion-area (OAR), y asi tener ######
#### aproximacion estrecha de la abundancia de un taxon.                                      #####
#### Kunin, W.E. (1998) Extrapolating species abundance across spatial scales. Science        #####
#### JIMMY JAPON 29/!2/1013  (jimmyluisjapon@gmail.com)                                       #####
###################################################################################################
#### librerias requeridas                           
library(raster)
library(sp)
library(spatstat)
library(shapefiles)
library(downscale)
library(terra)
library(sf)
########################################################################
# PRIMERO.- DATOS DE ENTRADA PRSENCIA Y POLIGONO DE LA ZONA DE ESTUDIO
#######################################################################
coor<-"C:/Users/Jimmy/Desktop/vizcacha_sig"# ruta de la carpeta donde se guardaran y tomaremos los archivos
setwd(coor)#definimos la ruta la ruta
getwd() # directorio estan los archivos espaciales
# leer archivos SIG  datos disponobles 
# la base de datos, esta disponible en https://github.com/jimmy-japon/dowscale-to-Lagidium-ahuacaense/blob/main/occs.csv
occs1 <- read.csv("C:/Users/Jimmy/Desktop/vizcacha_sig/occs.csv")# ruta donde esta la base de datos
head(occs1) #estructura de la tabla las columnas de longitud y latitud  deben constar
lagidium <- occs1[,5:6]
lagidium
g<-as.data.frame(lagidium)
# convertir los datos a objetos espaciales con una proyeccion UTM
recordsCoords <- SpatialPoints(data.frame(Lon = g$longitude,
                                          Lat = g$latitude), 
                               proj4string = CRS("+proj=longlat +datum=WGS84
                                                   +ellps=WGS84"))
coordsSp <- SpatialPoints(recordsCoords, proj4string = CRS("+proj=longlat +ellps=WGS84 +degrees=TRUE"))
coordsSp <- spTransform(coordsSp, CRS = CRS("+proj=cea +datum=WGS84 +units=km"))
plot(coordsSp) # ver los datos en nel espacio
# cargamos tambien nuestro extension de etudio esta disponoble en github
# https://github.com/jimmy-japon/dowscale-to-Lagidium-ahuacaense o tambiem
# https://drive.google.com/drive/folders/1ik90Wue1Qajw5YCnVWvGXAPeQPWiZpbI?usp=sharing
coastline <- vect("C:/Users/Jimmy/Desktop/vizcacha_sig/limiteszona/limiteszona.shp")# ruta del archivo
coastline <- project(coastline, "+proj=cea +datum=WGS84 +units=km") # proyecion
plot(coastline)
points(coordsSp, pch=16, col="red", cex=0.75)
############################################################################
# SEGUNDO.- GENERAR UNA GRILLA DE ENTRADA, PARAMETRIZAR LOS DATOS 
############################################################################
### generar una grilla de 2x2 km
cellWidth <- 2 #tamaño de grano en este caso 2x2km2
lag_raster <- raster(xmn = -8860.736,
                     xmx = -8822.736,
                     ymn = -523.7802,
                     ymx = -447.7802,
                     res = cellWidth,
                     crs = CRS("+proj=cea +datum=WGS84 +units=km"))
km2 <- rasterize(coordsSp, lag_raster, background = 0)
plot(km2)
km2[km2@data@values > 1] <- 1
writeRaster(km2, "km2.tif", NAflag=-999, overwrite=T)#guardar el raster en el directorio
km2
coastline
zona <- crop(coastline, km2, mask=TRUE)# cortamos los datos del raster con la extension
# es posible que se gener un error ciertas funciones del paquete de la funcion crop no responden
# es conveniente relizar el corte en un sig convencional en todo caso el archivo del recorte 
# esta disponoble adelante ------>  
####################################################################
# TERCERO.- REDUCCION DE LA ESCALA GENERAR CURVAS SCALA-AREA
####################################################################
# el archivo de las celdas 2x2 km2 esta en el siguiente enlace
# https://github.com/jimmy-japon/dowscale-to-Lagidium-ahuacaense/blob/main/lag_2x2.tif
grid<-rast("C:/Users/Jimmy/Desktop/vizcacha_sig/nuevos datos/lag_2x2.tif") # ruta del archivo
plot(grid)
points(coordsSp, pch=16, col="red", cex=0.75)
sum(values(grid), na.rm=TRUE) # ver celdas con presencias
terra::freq(grid)  # ver celdas con presencia y ausencias
# explorar umbrales para estandarizar datos de datos
thresh <- upgrain.threshold(atlas.data = grid, cell.width = 2,
                            scales = 3, thresholds = seq(0, 1, 0.01))
thresh$Thresholds # vemos los valores de los umbrales 
occupancy <- upgrain(grid, # el raster 
                     scales = 2, # tamaño de celda
                     method = "All_Sampled") # toda la area muestreada
# modelos y predicciones 
# modelo logistico
logis <- downscale(occupancies = occupancy, model = "Logis")
pred <- predict(logis, new.areas = c(0.25, 0.5, 1, 2, 4, 10, 25, 50))
pred$predicted # ver tabla de valores
# modelo INB
inb <- downscale(occupancies = occupancy,
                 model = "INB")
paramsNew <- list("C" = 0.1, "gamma" = 0.00001, "b" = 0.1) # especificar parametros 
inbNew <- downscale(occupancies = occupancy,
                    model = "INB",
                    starting_params = paramsNew)
inbPred <- predict(inb,
                   new.areas = c(0.25, 0.5, 1, 2, 4, 10, 25, 50),
                   plot = TRUE)
# modelo de thomas
thomas <- downscale(occupancies = occupancy,
                    model       = "Thomas",
                    tolerance   = 1e-3)
thomas.pred <- predict(thomas,
                       new.areas = c(0.25, 0.5, 1, 2, 4, 10, 25, 50),
                       tolerance = 1e-6)
# visualidar
par(mfrow=c(1,2))
logisPred <- predict(logis,
                     new.areas = areasPred,
                     plot = FALSE)
plot(thomas.pred)
plot(logisPred)
plot(inbPred)
# modelos ajustados 
ensemblelag <- ensemble.downscale(occupancy,
                                 models = "all",
                                 new.areas = c(0.25, 0.5, 1, 2, 4, 10, 25, 50),
                                 tolerance_mod = 1e-3,
                                 starting_params = list(INB = list(C = 10,
                                                                   gamma = 0.01,
                                                                   b = 0.1),
                                                        Thomas = list(rho = 1e-6,
                                                                      mu = 1,
                                                                      sigma = 1)))
ensemblelag$AOO[, c("Cell.area", "Means")]
#########################################################################################
####  VEAMOS LOS MISMOS MODELOS GENERADOR CON INFORMCION MANUAL DE SIG             ######
#########################################################################################
data<-read.table("clipboard", header=T)
attach(data)
occupancia <-data.frame(Cell.area,Ocupancy)
occupancia
logisMod3 <- downscale(occupancies = occupancia,
                      model       = "Logis",
                      extent      = 1642)
# modelo logistico
areasPred <- c(0.25, 0.5, 1, 2, 4, 8, 10, 25, 50)
logisPred3 <- predict(logisMod3,
                     new.areas = areasPred,
                     plot = FALSE)
plot(logisPred3)
# modelo INB
inb3 <- downscale(occupancies = occupancia,
                 model  = "INB",
                 extent = 1642)
inbPred3 <- predict(inb3,
                   new.areas = c(0.25, 0.5, 1, 2, 4, 10, 25, 50),
                   plot = TRUE)
# modelo de tomas
thomas3 <- downscale(occupancies = occupancia,
                    model       = "Thomas",
                    extent  =  1542,
                    tolerance   = 1e-3)
thomas.pred3 <- predict(thomas3,
                       new.areas = c(0.25, 0.5, 1, 2, 4, 10, 25, 50),
                       tolerance = 1e-6)
# visualizar
par(mfrow=c(1,3))
logisPred <- predict(logis,
                     new.areas = areasPred,
                     plot = FALSE)
plot(thomas.pred3)
plot(logisPred3)
plot(inbPred3)
# Graficar curvas escala-area
par(mfrow = c(1, 2))
plot(inbPred,
     col.pred = "blue",  # change the colour of the prediction
     pch      = 16,       # change point character
     lwd.obs  = 3)
legend("bottomright", legend = c("obs", "pred"),
       lwd = 3, col = c("black", "blue")) 
par(mfrow = c(1, 2))
plot(logisPred,
     col.pred = "blue",  # change the colour of the prediction
     pch      = 16,       # change point character
     lwd.obs  = 3)
legend("bottomright", legend = c("obs", "pred"),
       lwd = 3, col = c("black", "blue"))
plot(thomas.pred,
     col.pred = "blue",  # change the colour of the prediction
     pch      = 16,       # change point character
     lwd.obs  = 3)
legend("bottomright", legend = c("obs", "pred"),
       lwd = 3, col = c("black", "blue"))

# datos crudos de Sig
par(mfrow = c(1, 2))
plot(inbPred3,
     col.pred = "red",  # change the colour of the prediction
     pch      = 16,       # change point character
     lwd.obs  = 3)
legend("bottomright", legend = c("obs", "pred"),
       lwd = 3, col = c("black", "red")) 
plot(logisPred3,
     col.pred = "red",  # change the colour of the prediction
     pch      = 16,       # change point character
     lwd.obs  = 3)
legend("bottomright", legend = c("obs", "pred"),
       lwd = 3, col = c("black", "red"))
plot(thomas.pred3,
     col.pred = "red",  # change the colour of the prediction
     pch      = 16,       # change point character
     lwd.obs  = 3)
legend("bottomright", legend = c("obs", "pred"),
       lwd = 3, col = c("black", "red"))
