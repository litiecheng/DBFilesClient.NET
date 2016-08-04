# DBFilesClient.NET
A blazing-fast "new" DBC/DB2 reader for World of Warcraft.
Name is totally not stolen from [LordJZ's library](http://github.com/LordJZ/DBFilesClient.NET).

Built on top of [Sigil](https://github.com/kevin-montrose/Sigil).

## Planned features

1. Support more flavors of DB2 (WDB2, WDB3, WDB4)
2. Ultimately remove references to [Sigil](https://github.com/kevin-montrose/Sigil). It is a great library, but I only keep it because the code has a tendancy to encounter random errors during record loading. Debugging showed no difference in the IL generated by Sigil, and the standard .NET generator. Yet only Sigil's works.

## Usage

As well as exposing its standard `Storage<T>` type, the library exposes a `DBFileNameAttribute`. This attribute binds a file name to a structure. Use cases include loading a lot of files through reflection. The code below, snipped off the tests library, shows how to use this attribute.

```csharp
foreach (var type in Assembly.GetAssembly(typeof (ReaderTest)).GetTypes())
{
    if (!type.IsClass)
        continue;

    var attr = type.GetCustomAttribute<DBFileNameAttribute>();
    if (attr == null)
        continue;

    var instanceType = typeof (Storage<>).MakeGenericType(type);
    var instance = Activator.CreateInstance(instanceType, $@"{attr.FileName}.db2");
    ...
}
```

Regular instantiation goes by

```csharp
var instance = new Storage<AreaTriggerEntry>("AreaTrigger.db2");
```

Note that `Storage<T>` behaves like a `Dictionary<int, T>`.

### File formats and structures

For all file formats, you should provide reference-type structures to the library. Fields (and not properties, though that is subject to changes in the future) will get loaded in the order they are defined.

Bidimentional arrays are not supported; neither are collections.

#### WDBC

Nothing special here, just mark arrays with MarshalAsAttribute, specifying the size.

```c#
public sealed class AreaTableEntry
{
    [MarshalAs(UnManagedType.ByValArray, SizeConst = ...)]
    public uint[] Flags;
    public string ZoneName;
    public float AmbientMultiplier;
    public string AreaName;
    public ushort MapID;
    ...
}
```

#### WDB5

Due to the nature of the WDB5 format, arrays do not need *always* an explicit size defined through attributes. Size is inferred from the file.
Due to the code having difficulty to determine the size of arrays at the end of the record, you can make sure your array is properly loaded by decorating that field with a `MarshalAsAttribute`.


```c#
public sealed class AreaTableEntry
{
    public uint[] Flags;
    public string ZoneName;
    public float AmbientMultiplier;
    public string AreaName;
    public ushort MapID;
    ...
    // Optional, but use it if you want to be safe
    [MarshalAs(UnManagedType.ByValArray, SizeConst = ...)]
    public int[] Element;
}
```

Providing explicit array size for non-ending fields can also be done, but it will most likely be redundant.

## Performance

This section only refers to WDB5 DB2 files. 

Here are a few examples (randomly selected) of loading speeds for various files.
My work laptop has an i3-2310M CPU and 8Gb of DDR3 RAM.

Expected record count in the table below is calculated depending on field meta:
* If the file has an offset map, we count there.
* Otherwise, we use the record count in header.

We then add the amount of entries in the copy table.

```
File name                        Time to load        Record count   Expected   OK
---------------------------------------------------------------------------------
Spell                            00:00:12.3439573    158280         158280     OK
SpellEffect                      00:00:03.4151068    235970         235970     OK
Item-sparse                      00:00:02.5432846    93432          93432      OK
SoundKit                         00:00:01.9559884    66871          66871      OK
ItemSearchName                   00:00:01.9485236    76339          76339      OK
CreatureDisplayInfo              00:00:01.5684715    62121          62121      OK
SpellXSpellVisual                00:00:01.3541731    97038          97038      OK
CriteriaTree                     00:00:01.0931841    42868          42868      OK
TaxiPathNode                     00:00:01.0269692    69861          69861      OK
WMOAreaTable                     00:00:01.0084363    37874          37874      OK
ItemModifiedAppearance           00:00:00.9967128    79573          79573      OK
SpellMisc                        00:00:00.8513530    159202         159202     OK
SpellInterrupts                  00:00:00.3890539    46785          46785      OK
ItemSpecOverride                 00:00:00.3868698    55581          55581      OK
SpellCategories                  00:00:00.3722960    39087          39087      OK
Item                             00:00:00.2534158    117277         117277     OK
SpellCastTimes                   00:00:00.0534415    129            129        OK
```

## Reference

File specs for the various flavors of DB2 and DBC files can be found [here](http://wowdev.wiki/DBC).

## Thanks

In no particuliar order:
- #modcraft on QuakeNet.
- Kevin Montrose for [Sigil](https://github.com/kevin-montrose/Sigil).
