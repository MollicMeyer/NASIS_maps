# NASIS_maps
## Codes used to create soil property maps.
### Creates A, B, and C horizon thickness maps for any gSSURGO database. Code is not efficient at all because I had to meet certain requirements for GIS Programming and Automation class. Will Create a cleaner script.
### Requires 1) gSSURGO File Geodatabase; 2) an empty Template mxd file; 3) Symbology layer files in an mxd file for thickness rasters (line 255) 4) Script saved to workspace 

```
##Import modules
import arcpy, os, sys, datetime, traceback
import arcpy.mapping as mp
from arcpy.sa import *

# Check out the ArcGIS Spatial Analyst extension license
arcpy.CheckOutExtension("Spatial")

##Set up workspace and source variables
path = os.getcwd()
ssurgo = os.path.join(path, 'gSSURGO_IA.gdb')
maps = os.path.join(path, 'ssurgo_props.mxd')
dom_comp = os.path.join(ssurgo, 'dom_comp')
arcpy.env.workspace = ssurgo
arcpy.env.overwriteOutput = True

try:
    #set local variables
    CO = "component"
    expression = "majcompflag = 'Yes'"

    SMU = "MapunitRaster_10m"
    joinField1 = 'MUKEY'
    joinTable1 = 'mapunit'
    fieldList1 = ["muname", "musym"]

    ##Process runtime function
    def runtime(now, later):
        elapsed = later - now
        print(statement + str(elapsed))
        
    now = datetime.datetime.now()
    # Join the mapunit fields to SMU table
    arcpy.JoinField_management(SMU, joinField1, joinTable1, joinField1, fieldList1)
    print('Joining standard soil map unit attributes')
    later = datetime.datetime.now()
    statement = ('Attributes joined: ')
    runtime(now, later)
    del now
    del later
    
    
    #Set local variables for dominant component join
    joinTable2 = "dom_comp"
    compFields = ["compname", "comppct_r", "slope_r", "slopelenusle_r", "hydgrp", "cokey"]
    
    ##Select dominant components and export relevant component fields to new table
    now = datetime.datetime.now()
    print('Joining dominant components and basic attributes')
    arcpy.TableSelect_analysis(CO, joinTable2, expression)
    
    arcpy.JoinField_management(SMU, joinField1, joinTable2, joinField1, compFields)
    later = datetime.datetime.now()
    statement = ('Dominant component attributes joined: ')
    runtime(now, later)
    del now
    del later
    
    #Set local dominant component horizon table variables
    hor = 'chorizon'
    joinField2 = 'cokey'
    expression2 = "cokey_1 IS NOT NULL"
    expressionA = " \"desgnmaster\" LIKE 'A%' "
    expressionB = " \"desgnmaster\" LIKE 'B%' "
    expressionC = " \"desgnmaster\" LIKE 'C%' "
    dom_hor = "dom_hor"

    ##join dominant component codes to cHorizon table
    print('Joining dominant component horizons and basic attributes')
    now = datetime.datetime.now()
    arcpy.JoinField_management(hor, joinField2, joinTable2, joinField2, 'cokey')
    ##Select and export records that are NOT NULL; obtains dominant component horizons table
    arcpy.TableSelect_analysis(hor, dom_hor, expression2)

    ## Select only A, B, and C master horizons
    arcpy.TableSelect_analysis(dom_hor, "dom_horA", expressionA)
    arcpy.TableSelect_analysis(dom_hor, "dom_horB", expressionB)
    arcpy.TableSelect_analysis(dom_hor, "dom_horC", expressionC)

    dom_horA = "dom_horA"
    dom_horB = "dom_horB"
    dom_horC = "dom_horC"


    dom_horABC = [[dom_horA, "thickA"], [dom_horB, "thickB"], [dom_horC, "thickC"]]
    later = datetime.datetime.now()
    
    statement = ('Dominant component horizon attributes joined: ')
    runtime(now, later)
    del now
    del later
   
    ##Calculate Fields conditionally
    print('Querying horizon depths for each dominant component: Calculating total thickness')
    now = datetime.datetime.now()
    for items in dom_horABC:
        arcpy.Statistics_analysis(items[0], items[1], [["hzthk_r", "SUM"]], "cokey")
    del items
    later = datetime.datetime.now()
    
    statement = ('Thickness calculated for each component A, B, and C horizon: ')
    runtime(now, later)
    del now
    del later

    print('Creating empty tables for insertion')
    now = datetime.datetime.now()
    arcpy.CreateTable_management(ssurgo, "Atable", '', '')
    arcpy.CreateTable_management(ssurgo, "Btable", '', '')
    arcpy.CreateTable_management(ssurgo, "Ctable", '', '')

    tablelist = [["Atable", "cokeyA", "ThickA_cm"], ["Btable", "cokeyB", "ThickB_cm"],["Ctable", "cokeyC", "ThickC_cm"]]

    print 'adding join fields to empty table'
    for tables in tablelist:
        arcpy.AddField_management(tables[0], tables[1], "DOUBLE", 10, 10, "", "", "NULLABLE")
        arcpy.AddField_management(tables[0], tables[2], "DOUBLE", 10, 10, "", "", "NULLABLE")
    del tables
    
    later = datetime.datetime.now()
    statement = ('Empty table created and prepared for insertion: ')
    runtime(now, later)
    del now
    del later

    arcpy.TableSelect_analysis("thickA", "thickA_cm", "SUM_hzthk_r IS NOT NULL")
    arcpy.TableSelect_analysis("thickB", "thickB_cm", "SUM_hzthk_r IS NOT NULL")
    arcpy.TableSelect_analysis("thickC", "thickC_cm", "SUM_hzthk_r IS NOT NULL")
    
    print('Querying generated statistics table: Splitting into A, B, and C horizons')
    now = datetime.datetime.now()
    Alistc = []
    Alistm = []
    cursor = arcpy.da.SearchCursor("thickA_cm", ["cokey", "SUM_hzthk_r"])
    for row in cursor:
        Alistc.append(row[0])
        Alistm.append(row[1])

    del row
    del cursor

    Blistc = []
    Blistm = []
    cursor = arcpy.da.SearchCursor("thickB_cm", ["cokey", "SUM_hzthk_r"])
    for row in cursor:
        Blistc.append(row[0])
        Blistm.append(row[1])

    del row
    del cursor

    Clistc = []
    Clistm = []
    cursor = arcpy.da.SearchCursor("thickC_cm", ["cokey", "SUM_hzthk_r"])
    for row in cursor:
        Clistc.append(row[0])
        Clistm.append(row[1])
        
    del row
    del cursor
    
## Zip lists by Cokey and Thickness
    A = zip(Alistc,Alistm)
    B = zip(Blistc,Blistm)
    C = zip(Clistc,Clistm)
    alldata = zip(Alistm, Blistm, Clistm)
    later = datetime.datetime.now()
    
    statement = ('Query and horizon split complete: ')
    runtime(now, later)
    del now
    del later
    
    ##Function for Average of thickness List
    def Average(lst):
        return sum(lst) / len(lst)
    ##Function for Median of thickness List
    def Median(lst):
        n = len(lst)
        if n < 1:
                return None
        if n % 2 == 1:
                return sorted(lst)[n//2]
        else:
                return sum(sorted(lst)[n//2-1:n//2+1])/2.0
        
    for lists in alldata:
        average = Average(lists)
        median = Median(lists)
        print("Average thickness of list = " + str(average))
        print("Median thickness of list = " + str(median))
        
##Start editing empty Tables for InsertCursor
    print('Beginning edit session: InsertCursor Operation')
    now = datetime.datetime.now()
    edit = arcpy.da.Editor(ssurgo)
    edit.startEditing(False, False)
    
    #Start edit operation
    edit.startOperation()
    
    curs1 = arcpy.da.InsertCursor("Atable", ["cokeyA", "ThickA_cm"])
    for row in A:
        curs1.insertRow(row)
    del row
    del curs1
    curs2 = arcpy.da.InsertCursor("Btable", ["cokeyB", "ThickB_cm"])
    for row in B:
        curs2.insertRow(row)
    del row
    del curs2
    curs3 = arcpy.da.InsertCursor("Ctable", ["cokeyC", "ThickC_cm"])
    for row in C:
        curs3.insertRow(row)
    del row
    del curs3

    edit.stopOperation()
    edit.stopEditing(True)
    later = datetime.datetime.now()
    statement = ('Stop edit session: ')
    runtime(now, later)
    del now
    del later
    
#######################################################################################################################
    print 'Begin joining horizon data to gSSURGO raster attribute table'
    now = datetime.datetime.now()
 
    for fields in tablelist:
        arcpy.JoinField_management(SMU, "cokey", fields[0], fields[1], fields[2])
        outRaster = Lookup(SMU, fields[2])
        outRaster.save(os.path.join(path, str(fields[2]) + '.tif'))
        print('Joining thickness fields: ' + str(fields))

    later = datetime.datetime.now()
    statement = ('Join complete: ')
    runtime(now, later)
    del now
    del later
    
    print('Commencing genetic horizon thickness map generation')
    now = datetime.datetime.now()
    arcpy.env.workspace = path
    arcpy.env.overwriteOutput = True

    rasterlist = arcpy.ListFiles('*.tif')
    maps = os.path.join(path, 'Template.mxd')
    symmap = os.path.join(path, 'Sym_Template.mxd')
    sym = mp.MapDocument(symmap)
    df2 = mp.ListDataFrames(sym, 'Layers')[0]
    symbology = mp.ListLayers(sym, '', df2)
    titlelist = ["A Horizon Thickness", "B Horizon Thickness", "C Horizon Thickness"]
    symbology.reverse()

    zipper = zip(rasterlist, titlelist, symbology)

    for layers in zipper:
        soil_name = layers[0]
        mxd = mp.MapDocument(maps)
        df = mp.ListDataFrames(mxd)[0]
        print('Processing: ' + layers[0])
    # create temporary layer from given raster then add layer to map document
        soilLayer = arcpy.MakeRasterLayer_management(layers[0], soil_name[:-4])
        soilLayerFile = arcpy.SaveToLayerFile_management(soilLayer, str(soilLayer) + '.lyr')
        layerFileObj = mp.Layer(soilLayerFile.getOutput(0))
        mp.AddLayer(df, layerFileObj)
        updateLayer = mp.ListLayers(mxd, "", df)[0]
        mp.UpdateLayer(df, updateLayer, layers[2], True)

    #Change title
        titles = mp.ListLayoutElements(mxd, 'TEXT_ELEMENT')[0]
        titles.text = layers[1]
    #Export as pdf 
        print 'Creating pdf for ' + soil_name
        pdf_file = os.path.join(path, soil_name + '.pdf')
        mp.ExportToPDF(mxd, pdf_file)
        ## create empty PDF
    pdfs = arcpy.ListFiles('*.pdf')
    pdfDoc = mp.PDFDocumentCreate(os.path.join(path,'ThickABC.pdf'))
    ##append PDFs
    for pdf in pdfs:
        if os.path.exists(os.path.join(path, soil_name + '.pdf')) == True:
            pdfDoc.appendPages(pdf)
        else:
            print('PDF did not generate')
    pdfDoc.saveAndClose()
    del pdfDoc
            
    #clean-up locks
    del mxd
    later = datetime.datetime.now()
    statement = ('Map generation complete: ')
    runtime(now, later)
    del now
    del later
    
    
except Exception as err:
    print(err.args[0])
    exc_type, exc_value, exc_traceback = sys.exc_info()
    print "*** extract_tb:"
    print repr(traceback.extract_tb(exc_traceback))
    print "*** format_tb:"
    print repr(traceback.format_tb(exc_traceback))
    print "*** tb_lineno:", exc_traceback.tb_lineno

finally:
    print('Project Complete')


