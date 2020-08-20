JTS update from 1.14 to 1.17
============================

#### 1. Update Maven configuration (pom.xml): repositories

Remove two obsolete Maven repositories:

```xml
<repository>
    <id>maven central</id>
    <name>Central Maven 2 Repository</name>
    <url>http://central.maven.org/maven2</url>
</repository>

<repository>
    <id>cam</id>
    <name>Cambridge Maven 2 Repository</name>
    <url>http://maven.ch.cam.ac.uk/m2repo/</url>
</repository>
```

Add a new Maven repository:

```xml
<repository>
    <id>Maven Central group</id>
    <name>Maven Central group Repository</name>
    <url>https://mvnrepository.com/artifact/</url>
</repository>
```

Commit and push these changes:

```sh
git add .
git commit -m "Update Maven configuration" -m "Remove two obsolete repositories, and add a new one"
git push -u origin master
```


#### 2. Remove the WFS part
This includes the removal of the two following packages:
- `de.latlon.deejump.wfs`
- `org.deegree`

Modifications of the related classes consist mostly in commenting the
WFS code and the imports, and in updating a couple of `if` statements. The following classes require some modifications:
- `com.vividsolutions.jump.workbench.ui.LayerNameRenderer`
- `org.openjump.core.ui.plugin.layer.ExportLayerableEnvelopeAsGeometryPlugIn`
- `org.openjump.core.ui.plugin.layer.LayerableStylePlugIn`
- `org.openjump.core.ui.plugin.layer.NewLayerPropertiesPlugIn`
- `org.openjump.core.ui.plugin.raster.CropWarpPlugIn`
- `org.openjump.core.ui.plugin.task.Utils`

These changes are visible in this [commit](https://github.com/openjump-gis/openjump-migration/commit/e9e95e791484a9484bde2e5a864a685e32a9453b).

Commit and push these changes:

```sh
git add .
git commit -m "Remove the WFS functionalities" -m "Remove the following packages:
- de.latlon.deejump.wfs
- org.deegree

Modifications of the related classes consist mostly in commenting the
WFS code and the imports, and in updating a couple of if statements
(nothing sophisticated)."
git push -u origin master
```


#### 3. Update Maven configuration (pom.xml): JTS and deegree dependencies

Remove JTS 1.14 dependencies:
```xml
<dependency>
    <groupId>com.vividsolutions</groupId>
    <artifactId>jts-core</artifactId>
    <version>1.14.0</version>
</dependency>
<dependency>
    <groupId>com.vividsolutions</groupId>
    <artifactId>jts-io</artifactId>
    <version>1.14.0</version>
</dependency>
```

Remove deegree 2 (core) dependency:
```xml
<dependency>
    <groupId>org.deegree2</groupId>
    <artifactId>deegree2-core</artifactId>
    <version>2.6-pre2-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
```

Add JTS 1.17 dependencies:
```xml
<dependency>
    <groupId>org.locationtech.jts</groupId>
    <artifactId>jts-core</artifactId>
    <version>1.17.0</version>
</dependency>
<dependency>
    <groupId>org.locationtech.jts.io</groupId>
    <artifactId>jts-io-common</artifactId>
    <version>1.17.0</version>
</dependency>
```

Note that it would be better to add the JTS version as a global property and tp update each JTS dependency entry accordingly using `<version>${jts.version}</version>`:

```xml
<properties>
    ...
    <jts.version>1.17.0</jts.version>
    ...
</properties>
```

Here is a way to update automatically all the JTS related classes using the Eclipse IDE:
- Press `CTRL + H`, or use the `Search` menu, then `Search > File`,
- Click the `File Search Tab`,
- Use `com.vividsolutions.jts.` as `Containing text`,
- Use `\*.java` as `File name patterns (separated by comma)`,
- Use your current project as `Working set` for the `Scope`,
- Click the `Replace...` button at the bottom of the user interface,
- Replace `com.vividsolutions.jts.` with `org.locationtech.jts.`,
- The precomputed numbers indicate that there are 1456 matches in 549 files,
- Then click the `OK` button.

At this point, all classes should compile, except two:
- `com.vividsolutions.jump.geom.MakeValidOp`,
- `org.openjump.core.ui.plugin.tools.ReducePointsISAPlugIn`.

The `ReducePointsISAPlugIn` class is relatively easy to update, using the `PackedCoordinateSequenceFactory` class to replace `DefaultCoordinateSequenceFactory`.

Some updates have been made

The `MakeValidOp` class has been updated, cf. [changes](https://github.com/openjump-gis/openjump-migration/commit/3c24bce2bc6c69d2c786af5c9c0a4737b07666ad#diff-420fcf29102aedb5617766391802e29a). Despite the fact that it now compiles, there is still a problem with the 'nodePolygon(Polygon polygon)' method which replaces the Z values by NaN for 3D geometries. This also affects XYZM geometry type. If this problem can not be fixed before the final migration, an issue should be created.

Commit and push these changes:
```sh
git add .
git commit -m "Update JTS from 1.14 to 1.17 " -m "This update includes:
- remove deegree 2 core dependency,
- update JTS dependency,
- update JTS io (common) dependency,
- update all JTS related classes, including the renaming of all the
imports from com.vividsolutions.jts. to org.locationtech.jts,
- modify/update two classes which failed to compile due to
internal JTS changes between 1.14 and 1.17
(org.openjump.core.ui.plugin.tools.ReducePointsISAPlugIn and
com.vividsolutions.jump.geom.MakeValidOp).

Note: the 'nodePolygon(Polygon polygon)' method of the MakeValidOp
class replaces the Z values by NaN. This also affects XYZM geometry type."
git push -u origin master
```
