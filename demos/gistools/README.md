# GISTools 项目解读文档

> 基于开源GIS和Java的GIS二次开发库深度解析

## 目录

1. [项目概述](#项目概述)
2. [技术架构](#技术架构)
3. [核心数据模型](#核心数据模型)
4. [数据格式类型](#数据格式类型)
5. [GIS引擎策略](#gis引擎策略)
6. [几何处理工具](#几何处理工具)
7. [坐标系处理](#坐标系处理)
8. [数据转换核心代码](#数据转换核心代码)
9. [最佳实践与应用场景](#最佳实践与应用场景)

---

## 项目概述

GISTools是一个基于Java的GIS二次开发库，提供了统一的空间数据处理接口。项目整合了多个开源GIS库的能力，包括：

- **GeoTools 28.5**：Java领域最成熟的开源GIS工具包
- **JTS (Java Topology Suite) 1.20.0**：几何对象处理和空间关系计算
- **ESRI Geometry API 2.2.4**：ESRI几何引擎，提供高性能几何运算
- **GDAL/OGR 3.10.0**：强大的空间数据格式转换库

### 项目结构

```
gistools/
├── base/                           # 核心模块
│   └── src/main/java/com/southsmart/gistools/base/
│       ├── converter/              # 数据格式转换器
│       │   ├── GeometryConverter.java      # 要素级格式转换
│       │   └── WktLayerConverter.java      # 图层级格式转换
│       ├── enums/                  # 枚举定义
│       │   ├── DataFormatType.java         # 数据格式类型
│       │   ├── FieldDataType.java          # 字段数据类型
│       │   ├── GeometryType.java           # 几何类型
│       │   ├── GisEngineType.java          # GIS引擎类型
│       │   └── TopologyValidationErrorType.java  # 拓扑错误类型
│       ├── geometry/               # 几何处理工具
│       │   ├── ESRIGeometryUtil.java       # ESRI几何工具
│       │   └── JTSGeometryUtil.java        # JTS几何工具
│       ├── layer/                  # 图层数据模型
│       │   ├── txt/                        # 国土TXT格式
│       │   └── wkt/                        # WKT格式模型
│       ├── model/                  # 通用数据模型
│       └── util/                   # 工具类
│           ├── CrsUtil.java                # 坐标系工具
│           ├── OgrUtil.java                # GDAL/OGR工具
│           ├── PostgisUtil.java            # PostGIS工具
│           ├── ShpUtil.java                # Shapefile工具
│           ├── TxtLayerUtil.java           # TXT图层工具
│           └── ...
├── pom.xml                         # Maven配置
└── Dockerfile                      # Docker构建配置
```

---

## 技术架构

### 核心设计理念

GISTools采用**WktLayer**作为中间数据模型，实现不同GIS数据格式间的无缝转换：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Shapefile  │     │   GeoJSON   │     │   PostGIS   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────────────────────────────────────────────┐
│                     WktLayer                         │
│    (统一的中间数据模型：WKT几何 + 属性字段)           │
└──────────────────────────────────────────────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   FileGDB   │     │  国土TXT    │     │   其他格式  │
└─────────────┘     └─────────────┘     └─────────────┘
```

### 双引擎架构

项目实现了GeoTools和GDAL双引擎支持，根据数据格式和环境自动选择最优引擎：

```java
public enum GisEngineType {
    GEOTOOLS("GEOTOOLS"),      // GeoTools引擎
    GDAL("GDAL"),              // GDAL引擎
    AUTO("自动选择，GDAL优先");  // 自动选择

    public static GisEngineType getGisEngineType(GisEngineType gisEngineType) {
        if (gisEngineType == null || gisEngineType == AUTO) {
            if (Boolean.TRUE.equals(OgrUtil.getOgrInitSuccess())) {
                return GDAL;
            } else {
                return GEOTOOLS;
            }
        }
        return gisEngineType;
    }
}
```

---

## 核心数据模型

### WktLayer - 图层数据模型

WktLayer是整个框架的核心数据结构，它定义了一个完整的GIS图层：

```java
@Data
public class WktLayer implements Serializable {
    /**
     * 图层名称（英文名）
     */
    private String ywName;
    
    /**
     * 图层别名（中文名）
     */
    private String zwName;
    
    /**
     * 字段集合
     */
    private List<WktField> fields;
    
    /**
     * 要素集合
     */
    private List<WktFeature> features;
    
    /**
     * 空间参考WKID（如4490代表CGCS2000地理坐标系）
     */
    private Integer wkid;
    
    /**
     * 空间类型（点/线/面等）
     */
    private GeometryType geometryType;
    
    /**
     * 容差值（投影坐标系为0.0001，地理坐标系为0.000000001）
     */
    private Double tolerance;
}
```

### WktFeature - 要素数据模型

```java
@Data
public class WktFeature implements Serializable {
    /**
     * 要素ID
     * 注意：GDAL、GeoTools、ArcGIS等对FID/OID定义不同
     * 推荐使用属性过滤而非依赖此字段
     */
    private String wfId;
    
    /**
     * 要素图形WKT字符串
     * 例如：POLYGON((120.1 30.2, 120.2 30.2, 120.2 30.3, 120.1 30.2))
     */
    private String wkt;
    
    /**
     * 要素属性值列表
     */
    private List<WktFieldValue> fieldValues;
}
```

### WktField - 字段定义模型

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class WktField implements Serializable {
    /**
     * 字段名称（英文）
     */
    private String ywName;
    
    /**
     * 字段别名（中文）
     */
    private String zwName;
    
    /**
     * 字段描述
     */
    private String description;
    
    /**
     * 字段类型
     */
    private FieldDataType dataType;
}
```

### 字段数据类型

```java
@Getter
public enum FieldDataType {
    INTEGER(new int[]{0}, Integer.class, 0),
    DOUBLE(new int[]{2}, Double.class, 2),
    STRING(new int[]{1, 3, 4, 6, 5, 7, 13}, String.class, 4),
    BINARY(new int[]{8}, byte[].class, 8),
    DATE(new int[]{9}, LocalDate.class, 9),
    TIME(new int[]{10}, LocalTime.class, 10),
    DATETIME(new int[]{11}, LocalDateTime.class, 11),
    LONG(new int[]{12}, Long.class, 12);

    private final int[] gdalCodes;      // GDAL字段类型代码
    private final Class<?> typeClass;   // Java类型
    private final int defaultGdalCode;  // 默认GDAL代码
}
```

---

## 数据格式类型

### 支持的数据格式

GISTools支持多种GIS数据格式，每种格式对应不同的GDAL驱动：

```java
@Getter
public enum DataFormatType {
    WKT("WKT", null),                           // WKT字符串
    GEOJSON("GEOJSON", "GeoJSON"),              // GeoJSON格式
    ESRIJSON("ESRIJSON", "ESRIJSON"),           // ESRI JSON格式
    SHP("SHP文件", "ESRI Shapefile"),           // Shapefile格式
    TXT("国土TXT", null),                       // 国土部TXT格式
    FILEGDB("FILEGDB", "OpenFileGDB"),          // Esri FileGDB
    POSTGIS("POSTGIS", "PostgreSQL"),           // PostGIS数据库
    ARCSDE("ARCSDE", null);                     // ArcSDE（已废弃）

    private final String desc;              // 格式描述
    private final String gdalDriverName;    // GDAL驱动名称
}
```

### 几何类型定义

```java
@Getter
public enum GeometryType {
    POINT(0, "Point", Point.class, 1),
    MULTIPOINT(1, "MultiPoint", MultiPoint.class, 4),
    LINESTRING(2, "LineString", LineString.class, 2),
    LINEARRING(3, "LinearRing", LinearRing.class, 101),
    MULTILINESTRING(4, "MultiLineString", MultiLineString.class, 5),
    POLYGON(5, "Polygon", Polygon.class, 3),
    MULTIPOLYGON(6, "MultiPolygon", MultiPolygon.class, 6),
    GEOMETRYCOLLECTION(7, "GeometryCollection", GeometryCollection.class, 7);

    private final int typeCode;           // 类型代码
    private final String typeName;        // 类型名称
    private final Class<?> typeClass;     // JTS几何类
    private final int wkbGeometryType;    // WKB类型码
}
```

---

## GIS引擎策略

### 引擎选择逻辑

GISTools根据不同数据格式和运行环境，智能选择最合适的GIS引擎：

| 数据格式 | 推荐引擎 | 说明 |
|---------|---------|------|
| Shapefile | GeoTools/GDAL | 两者都支持，GDAL性能更优 |
| GeoJSON | GeoTools/GDAL | 两者都支持 |
| FileGDB | GDAL | GeoTools不支持，必须使用GDAL |
| PostGIS | GeoTools/GDAL | 两者都支持 |
| 国土TXT | 内置解析 | 自定义格式，无需外部引擎 |

### 引擎初始化检测

```java
public class OgrUtil {
    @Getter
    private static Boolean ogrInitSuccess;

    static {
        try {
            ogr.RegisterAll();
            gdal.SetConfigOption("GDAL_FILENAME_IS_UTF8", "YES");
            ogrInitSuccess = true;
        } catch (Throwable e) {
            ogrInitSuccess = false;
        }
    }

    public static void checkGdalEnv() {
        if (!Boolean.TRUE.equals(ogrInitSuccess)) {
            throw new RuntimeException("OGR初始化失败");
        }
    }
}
```

### 数据源操作示例

**GDAL数据源操作：**

```java
// 创建数据源
public static DataSource createDataSource(DataFormatType driverType, String path) {
    Driver driver = ogr.GetDriverByName(driverType.getGdalDriverName());
    return driver.CreateDataSource(path);
}

// 打开数据源
public static DataSource openDataSource(DataFormatType driverType, String path) {
    Driver driver = ogr.GetDriverByName(driverType.getGdalDriverName());
    return driver.Open(path, 1);
}

// 获取图层
public static Layer getLayer(DataSource dataSource, String layerName) {
    return dataSource.GetLayerByName(layerName);
}
```

---

## 几何处理工具

### JTS几何工具 (JTSGeometryUtil)

JTS是Java领域最常用的几何处理库，提供丰富的空间操作：

#### 基础几何操作

```java
public class JTSGeometryUtil {
    /**
     * 判断几何是否为空
     */
    public static boolean isEmpty(Geometry geom) {
        return geom.isEmpty();
    }

    /**
     * 获取几何长度
     */
    public static double length(Geometry geom) {
        return geom.getLength();
    }

    /**
     * 获取几何面积
     */
    public static double area(Geometry geom) {
        return geom.getArea();
    }

    /**
     * 获取几何中心点
     */
    public static Geometry centroid(Geometry geom) {
        return geom.getCentroid();
    }
}
```

#### 空间关系判断

```java
/**
 * 判断几何是否相交
 */
public static boolean intersects(Geometry a, Geometry b) {
    return a.intersects(b);
}

/**
 * 判断几何是否相离（无公共点）
 */
public static boolean disjoint(Geometry a, Geometry b) {
    return a.disjoint(b);
}

/**
 * 判断几何是否接触（有公共点无公共区域）
 */
public static boolean touches(Geometry a, Geometry b) {
    return a.touches(b);
}

/**
 * 判断几何是否包含
 */
public static boolean contains(Geometry a, Geometry b) {
    return a.contains(b);
}

/**
 * 判断几何是否在内部
 */
public static boolean within(Geometry a, Geometry b) {
    return a.within(b);
}

/**
 * 判断几何是否重叠
 */
public static boolean overlaps(Geometry a, Geometry b) {
    return a.overlaps(b);
}
```

#### 几何运算

```java
/**
 * 获取几何缓冲区
 */
public static Geometry buffer(Geometry geom, double distance) {
    return geom.buffer(distance);
}

/**
 * 获取几何凸包
 */
public static Geometry convexHull(Geometry geom) {
    ConvexHull ch = new ConvexHull(geom);
    return ch.getConvexHull();
}

/**
 * 获取几何交集
 */
public static Geometry intersection(Geometry a, Geometry b) {
    return a.intersection(b);
}

/**
 * 获取几何并集
 */
public static Geometry union(Geometry... geoms) {
    Geometry result = null;
    for (Geometry g : geoms) {
        if (result == null) {
            result = g;
        } else {
            result = result.union(g);
        }
    }
    return result;
}

/**
 * 获取差集（A - B）
 */
public static Geometry difference(Geometry a, Geometry b) {
    return a.difference(b);
}

/**
 * 获取对称差（并集 - 交集）
 */
public static Geometry symDifference(Geometry a, Geometry b) {
    return a.symDifference(b);
}
```

#### 拓扑验证

```java
/**
 * 判断几何拓扑是否合法
 */
public static IsValidModel isValid(Geometry geom) {
    IsValidOp isValidOp = new IsValidOp(geom);
    if (!isValidOp.isValid()) {
        TopologyValidationErrorType type = TopologyValidationErrorType
            .getByErrorType(isValidOp.getValidationError().getErrorType());
        String msg = type != null ? type.getDesc() : "未知拓扑错误";
        return new IsValidModel(false,
                isValidOp.getValidationError().getCoordinate(), 
                type, msg);
    }
    return new IsValidModel(true, null, null, null);
}

/**
 * 修复无效几何
 */
public static Geometry validate(Geometry geom) {
    if (geom instanceof Polygon || geom instanceof MultiPolygon) {
        if (geom.isValid()) {
            geom.normalize();
            return geom;
        }
        Polygonizer polygonizer = new Polygonizer();
        // 通过多边形化方式修复
        addPolygon((Polygon) geom, polygonizer);
        return toPolygonGeometry(polygonizer.getPolygons());
    }
    return geom;
}
```

### ESRI几何工具 (ESRIGeometryUtil)

ESRI几何API提供了另一套高性能的几何处理方案：

```java
public class ESRIGeometryUtil {
    /**
     * 计算几何距离
     */
    public static double distance(String awkt, String bwkt, Integer wkid) {
        Geometry a = EsriUtil.createGeometryByWkt(awkt);
        Geometry b = EsriUtil.createGeometryByWkt(bwkt);
        SpatialReference sr = SpatialReference.create(wkid);
        return GeometryEngine.distance(a, b, sr);
    }

    /**
     * 计算缓冲区
     */
    public static String buffer(String wkt, Integer wkid, double distance) {
        Geometry geom = EsriUtil.createGeometryByWkt(wkt);
        SpatialReference sr = SpatialReference.create(wkid);
        Geometry buffer = OperatorBuffer.local().execute(geom, sr, distance, null);
        return EsriUtil.getWktStr(buffer);
    }

    /**
     * 计算并集
     */
    public static String union(List<String> wkts, Integer wkid) {
        Geometry[] geoms = wkts.stream()
            .map(EsriUtil::createGeometryByWkt)
            .toArray(Geometry[]::new);
        SpatialReference sr = SpatialReference.create(wkid);
        Geometry union = GeometryEngine.union(geoms, sr);
        return EsriUtil.getWktStr(union);
    }

    /**
     * 简化几何（去除冗余节点）
     */
    public static String simplify(String wkt, Integer wkid) {
        Geometry geom = EsriUtil.createGeometryByWkt(wkt);
        SpatialReference sr = SpatialReference.create(wkid);
        Geometry simplified = OperatorSimplifyOGC.local().execute(geom, sr, false, null);
        return EsriUtil.getWktStr(simplified);
    }
}
```

### 拓扑错误类型

```java
@Getter
public enum TopologyValidationErrorType {
    ERROR(0, "拓扑检查错误"),
    REPEATED_POINT(1, "点重叠"),
    HOLE_OUTSIDE_SHELL(2, "洞在图形外"),
    NESTED_HOLES(3, "洞重叠"),
    DISCONNECTED_INTERIOR(4, "图形内部不连通"),
    SELF_INTERSECTION(5, "自相交"),
    RING_SELF_INTERSECTION(6, "环自相交"),
    NESTED_SHELLS(7, "图形重叠"),
    DUPLICATE_RINGS(8, "环重复"),
    TOO_FEW_POINTS(9, "点太少无法构成有效几何"),
    INVALID_COORDINATE(10, "无效坐标"),
    RING_NOT_CLOSED(11, "环未闭合");

    private final int errorType;
    private final String desc;
}
```

---

## 坐标系处理

### 支持的坐标系

GISTools重点支持中国常用的CGCS2000坐标系：

| EPSG代码 | 坐标系类型 | 说明 |
|---------|----------|------|
| 4490 | 地理坐标系 | CGCS2000经纬度 |
| 4491-4554 | 投影坐标系 | CGCS2000高斯投影（3度带） |

### 坐标系工具核心代码

```java
public class CrsUtil {
    private static Map<Integer, CoordinateReferenceSystem> supportedCRSList;

    /**
     * 获取支持的投影坐标系列表
     */
    private static Map<Integer, CoordinateReferenceSystem> supportedCRSList() {
        if (supportedCRSList != null && !supportedCRSList.isEmpty()) {
            return supportedCRSList;
        }

        supportedCRSList = new HashMap<>();
        for (int i = 4490; i < 4555; i++) {
            supportedCRSList.put(i, CRS.decode("EPSG:" + i, true));
        }
        return supportedCRSList;
    }

    /**
     * 获取坐标系容差
     * 投影坐标系：0.0001米
     * 地理坐标系：0.000000001度
     */
    public static double getTolerance(CoordinateReferenceSystem crs) {
        if (isProjectedCRS(crs)) {
            return 0.0001;
        } else {
            return 0.000000001;
        }
    }

    /**
     * 根据几何计算所在带号
     */
    public static int getDh(Geometry geometry) {
        Point point = geometry.getCentroid();
        int dh = 0;
        if (point.getX() < 180) {
            // 经纬度坐标
            dh = (int) ((point.getX() + 1.5) / 3);
        } else if (point.getX() / 10000000 > 3) {
            // 投影坐标（带号在X坐标首位）
            dh = (int) (point.getX() / 1000000);
        }
        return dh;
    }

    /**
     * 获取投影坐标系WKID
     * CGCS2000 3度带WKID = 4488 + 带号
     */
    public static Integer getProjectedWkid(int dh) {
        return 4488 + dh;
    }
}
```

### 坐标转换

```java
/**
 * 几何对象坐标系转换
 */
public static Geometry transform(Geometry geometry, Integer sourceWkid, Integer targetWkid) {
    if (sourceWkid.equals(targetWkid)) {
        return geometry;
    }

    CoordinateReferenceSystem sourceCRS = getSupportedCRS(sourceWkid).getValue();
    CoordinateReferenceSystem targetCRS = getSupportedCRS(targetWkid).getValue();
    
    MathTransform transform = CRS.findMathTransform(sourceCRS, targetCRS);
    return JTS.transform(geometry, transform);
}

/**
 * WktLayer坐标系转换
 */
public static WktLayer reproject(WktLayer wktLayer, Integer targetWkid) {
    wktLayer.check();
    WktLayer clone = ObjectUtil.cloneByStream(wktLayer);
    
    if (clone.getWkid().equals(targetWkid)) {
        return clone;
    }

    for (WktFeature feature : clone.getFeatures()) {
        Geometry geometry = new WKTReader().read(feature.getWkt());
        Geometry targetGeometry = transform(geometry, wktLayer.getWkid(), targetWkid);
        feature.setWkt(targetGeometry.toText());
    }

    clone.setWkid(targetWkid);
    clone.setTolerance(getTolerance(targetWkid));
    return clone;
}
```

---

## 数据转换核心代码

### 要素级格式转换 (GeometryConverter)

提供WKT、GeoJSON、EsriJSON、JTS Geometry之间的相互转换：

```java
public class GeometryConverter {
    /**
     * WKT转JTS Geometry
     */
    public static Geometry wkt2Geometry(String wkt) {
        WKTReader2 reader = new WKTReader2();
        return reader.read(wkt);
    }

    /**
     * WKT转GeoJSON
     */
    public static String wkt2Geojson(String wkt) {
        Geometry geometry = wkt2Geometry(wkt);
        return geometry2Geojson(geometry);
    }

    /**
     * WKT转EsriJSON
     */
    public static String wkt2EsriJson(String wkt, int wkid) {
        com.esri.core.geometry.Geometry geometry = EsriUtil.createGeometryByWkt(wkt);
        return EsriUtil.getEsriJson(wkid, geometry);
    }

    /**
     * GeoJSON转JTS Geometry
     */
    public static Geometry geojson2Geometry(String geojson) {
        GeometryJSON gjson = new GeometryJSON(16);
        return gjson.read(new StringReader(geojson));
    }

    /**
     * JTS Geometry转GeoJSON
     */
    public static String geometry2Geojson(Geometry geometry) {
        GeometryJSON gjson = new GeometryJSON(16);
        StringWriter writer = new StringWriter();
        gjson.write(geometry, writer);
        return writer.toString();
    }

    /**
     * EsriJSON转WKT
     */
    public static String esriJson2Wkt(String esrijson) {
        com.esri.core.geometry.Geometry geometry = EsriUtil.createGeometryByJson(esrijson);
        return EsriUtil.getWktStr(geometry);
    }
}
```

### 图层级格式转换 (WktLayerConverter)

#### Shapefile读取

```java
/**
 * Shapefile转WktLayer
 * 
 * @param shpPath          Shapefile路径
 * @param attributeFilter  属性过滤条件（SQL WHERE子句）
 * @param spatialFilterWkt 空间过滤条件（WKT几何）
 * @param gisEngineType    GIS引擎类型
 */
public static WktLayer fromShapefile(String shpPath, String attributeFilter, 
        String spatialFilterWkt, GisEngineType gisEngineType) {
    
    // 检测Shapefile编码
    Charset shpCharset = ShpUtil.check(shpPath);
    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        // GDAL方式读取
        gdal.SetConfigOption("SHAPE_ENCODING", shpCharset.name());
        String shpDir = FileUtil.getParent(shpPath, 1);
        String shpName = FileUtil.mainName(shpPath);
        return OgrUtil.layer2WktLayer(DataFormatType.SHP, shpDir, shpName, 
                attributeFilter, spatialFilterWkt);
    } else {
        // GeoTools方式读取
        File file = new File(shpPath);
        ShapefileDataStore shpDataStore = new ShapefileDataStore(file.toURI().toURL());
        shpDataStore.setCharset(shpCharset);
        String typeName = shpDataStore.getTypeNames()[0];
        SimpleFeatureSource source = shpDataStore.getFeatureSource(typeName);
        SimpleFeatureCollection collection = GeotoolsUtil.filter(source, 
                attributeFilter, spatialFilterWkt);
        shpDataStore.dispose();
        return fromSimpleFeatureCollection(collection);
    }
}
```

#### Shapefile写入

```java
/**
 * WktLayer转Shapefile
 */
public static void toShapefile(WktLayer wktLayer, String shpPath, GisEngineType gisEngineType) {
    wktLayer.check();
    EsriUtil.excludeSpecialFields(wktLayer.getFields());
    ShpUtil.formatFieldName(wktLayer.getFields());  // Shapefile字段名限制10字符

    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        gdal.SetConfigOption("SHAPE_ENCODING", "");
        Vector options = new Vector();
        options.add("ENCODING=UTF-8");
        String shpDir = FileUtil.getParent(shpPath, 1);
        String shpName = FileUtil.mainName(shpPath);
        OgrUtil.wktLayer2Layer(DataFormatType.SHP, shpDir, wktLayer, shpName, options);
    } else {
        // GeoTools方式写入
        File shapeFile = new File(shpPath);
        SimpleFeatureCollection featureCollection = toSimpleFeatureCollection(wktLayer);
        
        Map<String, Serializable> params = new HashMap<>();
        params.put(ShapefileDataStoreFactory.URLP.key, shapeFile.toURI().toURL());
        
        ShapefileDataStore ds = (ShapefileDataStore) 
            new ShapefileDataStoreFactory().createNewDataStore(params);
        ds.createSchema(featureCollection.getSchema());
        ds.setCharset(StandardCharsets.UTF_8);
        
        // 写入要素...
        
        // 创建CPG文件指定UTF-8编码
        String cpgPath = shpPath.substring(0, shpPath.lastIndexOf(".")) + ".cpg";
        FileUtil.writeString("UTF-8", cpgPath, StandardCharsets.UTF_8);
    }
}
```

#### GeoJSON转换

```java
/**
 * GeoJSON转WktLayer
 */
public static WktLayer fromGeoJSON(String geojsonPath, GisEngineType gisEngineType) {
    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        String layerName = FileUtil.mainName(geojsonPath);
        return OgrUtil.layer2WktLayer(DataFormatType.GEOJSON, geojsonPath, 
                layerName, null, null);
    } else {
        // GeoTools方式
        File file = new File(geojsonPath);
        Charset encoding = EncodingUtil.getFileEncoding(file);
        String geojsonString = FileUtil.readString(file, encoding);

        GeometryJSON gjson = new GeometryJSON(16);
        FeatureJSON fjson = new FeatureJSON(gjson);

        SimpleFeatureType featureType = fjson.readFeatureCollectionSchema(geojsonString, true);
        ListFeatureCollection collection = new ListFeatureCollection(featureType);
        
        try (FeatureIterator<SimpleFeature> features = fjson.streamFeatureCollection(geojsonString)) {
            while (features.hasNext()) {
                collection.add(features.next());
            }
        }

        return fromSimpleFeatureCollection(collection);
    }
}

/**
 * WktLayer转GeoJSON
 */
public static void toGeoJSON(WktLayer wktLayer, String geojsonPath, GisEngineType gisEngineType) {
    wktLayer.check();
    EsriUtil.excludeSpecialFields(wktLayer.getFields());

    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        String layerName = FileUtil.mainName(geojsonPath);
        OgrUtil.wktLayer2Layer(DataFormatType.GEOJSON, geojsonPath, wktLayer, layerName, null);
    } else {
        SimpleFeatureCollection featureCollection = toSimpleFeatureCollection(wktLayer);
        GeometryJSON gjson = new GeometryJSON(16);
        FeatureJSON fjson = new FeatureJSON(gjson);
        
        String geojsonString = fjson.toString(featureCollection);
        FileUtil.writeString(geojsonString, geojsonPath, "utf-8");
    }
}
```

#### FileGDB处理（仅GDAL）

```java
/**
 * FileGDB图层转WktLayer
 * 注意：FileGDB必须使用GDAL引擎
 */
public static WktLayer fromFileGDB(String gdbPath, String layerName, 
        String attributeFilter, String spatialFilterWkt, GisEngineType gisEngineType) {
    
    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        return OgrUtil.layer2WktLayer(DataFormatType.FILEGDB, gdbPath, 
                layerName, attributeFilter, spatialFilterWkt);
    } else {
        throw new RuntimeException("GDB数据源需要GDAL支持");
    }
}

/**
 * WktLayer转FileGDB
 */
public static void toFileGDB(WktLayer wktLayer, String gdbPath, 
        String featureDataset, String layerName, GisEngineType gisEngineType) {
    
    wktLayer.check();
    EsriUtil.excludeSpecialFields(wktLayer.getFields());

    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        Vector options = null;
        if (CharSequenceUtil.isNotBlank(featureDataset)) {
            options = new Vector();
            options.add("FEATURE_DATASET=" + featureDataset);
        }
        OgrUtil.wktLayer2Layer(DataFormatType.FILEGDB, gdbPath, wktLayer, layerName, options);
    } else {
        throw new RuntimeException("GDB数据源需要GDAL支持");
    }
}
```

#### PostGIS数据库操作

```java
/**
 * PostGIS图层转WktLayer
 */
public static WktLayer fromPostGIS(DbConnBaseModel dbConnBaseModel, String layerName, 
        String attributeFilter, String spatialFilterWkt, GisEngineType gisEngineType) {
    
    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        return OgrUtil.layer2WktLayer(DataFormatType.POSTGIS, 
                PostgisUtil.toGdalPostgisConnStr(dbConnBaseModel), 
                layerName, attributeFilter, spatialFilterWkt);
    } else {
        // GeoTools方式
        DataStore dataStore = PostgisUtil.getPostgisDataStore(dbConnBaseModel);
        SimpleFeatureSource source = dataStore.getFeatureSource(layerName);
        SimpleFeatureCollection collection = GeotoolsUtil.filter(source, 
                attributeFilter, spatialFilterWkt);
        dataStore.dispose();
        return fromSimpleFeatureCollection(collection);
    }
}

/**
 * WktLayer转PostGIS（支持批量写入）
 */
public static void toPostGIS(WktLayer wktLayer, DbConnBaseModel dbConnBaseModel, 
        String layerName, GisEngineType gisEngineType) {
    
    wktLayer.check();
    EsriUtil.excludeSpecialFields(wktLayer.getFields());

    gisEngineType = GisEngineType.getGisEngineType(gisEngineType);
    
    if (gisEngineType == GisEngineType.GDAL) {
        OgrUtil.wktLayer2Layer4Postgis(DataFormatType.POSTGIS, dbConnBaseModel, 
                wktLayer, layerName, null);
    } else {
        // GeoTools方式（支持多线程批量写入）
        SimpleFeatureCollection featureCollection = toSimpleFeatureCollection(wktLayer);
        JDBCDataStore dataStore = PostgisUtil.getPostgisDataStore(dbConnBaseModel);
        
        // 如果表不存在则创建
        if (!Arrays.asList(dataStore.getTypeNames()).contains(layerName)) {
            SimpleFeatureTypeBuilder tb = new SimpleFeatureTypeBuilder();
            tb.init(featureCollection.getSchema());
            tb.setName(layerName);
            dataStore.createSchema(tb.buildFeatureType());
        }

        // 批量写入（每批1000条，多线程并行）
        int batchSize = 1000;
        int count = features.size() / batchSize;
        ExecutorService executorService = ThreadUtil.newExecutor(count);
        
        for (int i = 0; i <= count; i++) {
            List<SimpleFeature> subList = features.subList(i * batchSize, 
                    Math.min((i + 1) * batchSize, features.size()));
            
            executorService.execute(() -> {
                // 写入数据库...
            });
        }
        
        executorService.shutdown();
        executorService.awaitTermination(Long.MAX_VALUE, TimeUnit.NANOSECONDS);
    }
}
```

#### 国土TXT格式处理

国土TXT是自然资源部规定的宗地坐标数据交换格式：

```java
/**
 * TXT文件格式示例：
 * 
 * [属性描述]
 * 格式版本号=
 * 数据产生单位=自然资源部
 * 数据产生日期=2024-01-01
 * 坐标系=2000国家大地坐标系
 * 几度分带=3
 * 投影类型=高斯克吕格
 * 计量单位=米
 * 带号=39
 * 精度=0.01
 * 转换参数=0,0,0,0,0,0,0
 * [地块坐标]
 * 4,0.1234,地块编号1,地块名称1,面,图幅号,用途,地类编码,图斑类型,地类,等别前,等别后,备注,@
 * 1,1,3849123.456,39456789.012
 * 2,1,3849234.567,39456890.123
 * 3,1,3849345.678,39456901.234
 * 4,1,3849123.456,39456789.012
 */

/**
 * 加载国土TXT文件
 */
public static TxtLayer loadTxt(String txtPath) {
    File file = new File(txtPath);
    Charset encoding = EncodingUtil.getFileEncoding(file);
    List<String> txtLines = FileUtil.readLines(file, encoding);
    
    // 解析[属性描述]和[地块坐标]两个模块
    TxtLayer txtLayer = new TxtLayer();
    
    // 解析属性描述
    // txtLayer.setGsbbh(...)  格式版本号
    // txtLayer.setSjscdw(...) 数据产生单位
    // txtLayer.setSjscrq(...) 数据产生日期
    // txtLayer.setZbx(...)    坐标系
    // txtLayer.setJdfd(...)   几度分带
    // txtLayer.setTylx(...)   投影类型
    // txtLayer.setJldw(...)   计量单位
    // txtLayer.setDh(...)     带号
    // txtLayer.setJd(...)     精度
    // txtLayer.setZhcs(...)   转换参数
    
    // 解析地块坐标
    // 属性行以@结尾，后续为坐标点
    // 坐标格式：点号,圈号,Y坐标,X坐标
    
    return txtLayer;
}

/**
 * TXT转WktLayer
 */
public static WktLayer parseTxtLayerToWktLayer(TxtLayer txtLayer, List<WktField> wktFields) {
    // 默认字段定义
    if (wktFields == null) {
        wktFields = new ArrayList<>();
        wktFields.add(new WktField("JZDS", "界址点数", null, FieldDataType.STRING));
        wktFields.add(new WktField("DKMJ", "地块面积", null, FieldDataType.STRING));
        wktFields.add(new WktField("DKBH", "地块编号", null, FieldDataType.STRING));
        wktFields.add(new WktField("DKMC", "地块名称", null, FieldDataType.STRING));
        // ... 更多字段
    }

    WktLayer wktLayer = new WktLayer();
    wktLayer.setYwName("TXT");
    wktLayer.setZwName(txtLayer.getName());
    
    // 根据带号计算WKID：CGCS2000 3度带 = 4488 + 带号
    int wkid = 4488 + NumberUtil.parseInt(txtLayer.getDh());
    wktLayer.setWkid(wkid);
    wktLayer.setGeometryType(GeometryType.MULTIPOLYGON);
    
    // 构建几何
    for (TxtFeature txtFeature : txtLayer.getFeatures()) {
        // 按圈号分组坐标点，构建Polygon
        // 第一圈为外环，其他圈为内环（洞）
        LinkedHashMap<Integer, List<Coordinate>> coordinateMap = new LinkedHashMap<>();
        for (TxtCoordinate coord : txtFeature.getCoordinates()) {
            coordinateMap.computeIfAbsent(coord.getQh(), k -> new ArrayList<>())
                .add(new Coordinate(coord.getX(), coord.getY()));
        }
        
        // 创建Polygon
        GeometryFactory gf = new GeometryFactory();
        LinearRing shell = gf.createLinearRing(coordinateMap.get(1).toArray(new Coordinate[0]));
        LinearRing[] holes = ...;  // 内环
        Polygon polygon = gf.createPolygon(shell, holes);
        
        WktFeature wktFeature = new WktFeature();
        wktFeature.setWkt(polygon.toText());
        // 设置属性值...
    }
    
    return wktLayer;
}
```

---

## 最佳实践与应用场景

### 1. 数据格式转换

**场景**：将Shapefile转换为GeoJSON供Web端使用

```java
// 读取Shapefile
WktLayer wktLayer = WktLayerConverter.fromShapefile(
    "/data/input.shp", 
    null,           // 无属性过滤
    null,           // 无空间过滤
    GisEngineType.AUTO
);

// 转换坐标系（投影坐标转经纬度）
if (wktLayer.getWkid() != 4490) {
    wktLayer = CrsUtil.reproject(wktLayer, 4490);
}

// 输出GeoJSON
WktLayerConverter.toGeoJSON(wktLayer, "/data/output.geojson", GisEngineType.AUTO);
```

### 2. 空间查询与筛选

**场景**：从PostGIS中查询指定区域内的数据

```java
DbConnBaseModel conn = new DbConnBaseModel();
conn.setDbtype("postgis");
conn.setHost("localhost");
conn.setPort("5432");
conn.setDatabase("gisdb");
conn.setSchema("public");
conn.setUser("postgres");
conn.setPasswd("password");

// 查询区域的WKT
String spatialFilter = "POLYGON((120.1 30.1, 120.2 30.1, 120.2 30.2, 120.1 30.2, 120.1 30.1))";

// 属性条件
String attributeFilter = "landuse = '农业'";

// 执行查询
WktLayer result = WktLayerConverter.fromPostGIS(
    conn, 
    "parcels",      // 图层名
    attributeFilter,
    spatialFilter,
    GisEngineType.AUTO
);
```

### 3. 几何分析

**场景**：计算多个地块的并集并验证拓扑

```java
// 读取数据
WktLayer wktLayer = WktLayerConverter.fromShapefile(...);

// 收集所有几何
List<Geometry> geometries = new ArrayList<>();
for (WktFeature feature : wktLayer.getFeatures()) {
    Geometry geom = GeometryConverter.wkt2Geometry(feature.getWkt());
    geometries.add(geom);
}

// 计算并集
Geometry union = JTSGeometryUtil.union(geometries.toArray(new Geometry[0]));

// 验证拓扑
IsValidModel validResult = JTSGeometryUtil.isValid(union);
if (!validResult.isValid()) {
    System.out.println("拓扑错误：" + validResult.getMessage());
    System.out.println("错误位置：" + validResult.getCoordinate());
    
    // 尝试修复
    union = JTSGeometryUtil.validate(union);
}

// 计算面积
double area = JTSGeometryUtil.area(union);
System.out.println("总面积：" + area + " 平方米");
```

### 4. 批量数据入库

**场景**：将大量Shapefile批量导入PostGIS

```java
List<File> shpFiles = FileUtil.loopFiles("/data/shapefiles/", file -> 
    file.getName().endsWith(".shp"));

for (File shpFile : shpFiles) {
    try {
        // 读取Shapefile
        WktLayer wktLayer = WktLayerConverter.fromShapefile(
            shpFile.getAbsolutePath(), null, null, GisEngineType.AUTO);
        
        // 图层名使用文件名
        String layerName = FileUtil.mainName(shpFile);
        
        // 写入PostGIS（支持追加）
        WktLayerConverter.toPostGIS(wktLayer, conn, layerName, GisEngineType.AUTO);
        
        System.out.println("已导入：" + layerName);
    } catch (Exception e) {
        System.err.println("导入失败：" + shpFile.getName() + " - " + e.getMessage());
    }
}
```

### 5. 国土数据处理

**场景**：处理自然资源部标准TXT格式数据

```java
// 加载TXT
WktLayer wktLayer = WktLayerConverter.fromTxtLayer(
    "/data/宗地.txt",
    null  // 使用默认字段定义
);

// 验证数据
for (WktFeature feature : wktLayer.getFeatures()) {
    Geometry geom = GeometryConverter.wkt2Geometry(feature.getWkt());
    IsValidModel valid = JTSGeometryUtil.isValid(geom);
    if (!valid.isValid()) {
        System.out.println("地块 " + feature.getValue("DKBH") + " 拓扑错误：" + valid.getMessage());
    }
}

// 导出为Shapefile
WktLayerConverter.toShapefile(wktLayer, "/data/output.shp", GisEngineType.AUTO);

// 或导出回TXT格式
TxtLayer msInfo = new TxtLayer();
msInfo.setSjscdw("XX国土局");
msInfo.setSjscrq("2024-01-01");

WktLayerConverter.toTxtLayer(
    wktLayer,
    "/data/output.txt",
    msInfo,
    null,  // 使用默认字段顺序
    39     // 带号
);
```

---

## 总结

GISTools通过以下设计实现了高效的GIS数据处理：

1. **统一数据模型**：WktLayer作为中间格式，实现多格式互转
2. **双引擎架构**：GeoTools和GDAL互补，灵活适应不同场景
3. **完整的几何处理**：JTS和ESRI双引擎提供丰富的空间分析能力
4. **坐标系支持**：重点支持中国CGCS2000系列坐标系
5. **批量处理优化**：多线程支持，适合大数据量处理

项目适用于：
- 国土资源数据处理
- 空间数据格式转换
- GIS数据ETL流程
- PostGIS数据管理
- 空间分析应用开发

---

*文档生成时间：2024*
*项目地址：https://github.com/znlgis/gistools*
