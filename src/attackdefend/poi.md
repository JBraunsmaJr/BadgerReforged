# Point of Interest
Similar to `SCR_Badger_SpawnArea` we have a POI. Primary difference is this isn't meant for `spawning`, but instead is a `CaptureArea`.

```csharp
[EntityEditorProps(category: "GameScripted/GameMode/Badger", description: "Defines a point of interest for Badger-Systems")]
class SCR_Badger_POIClass : SCR_CaptureAreaClass
{
}

class SCR_Badger_POI : SCR_CaptureArea
{
	protected override void OnInit(IEntity owner)
	{
		super.OnInit(owner);
		
		if(!GetGame().InPlayMode())
			return;
		
		SCR_Badger_BaseGameMode badger = SCR_Badger_BaseGameMode.GetInstance();
		
		if(!badger)
		{
			Print("[SCR_Badger_POI] <OnInit>: Was unable to find SCR_Badger_BaseGameMode, functionality will be limited", LogLevel.ERROR);
			return;
		}
		
		badger.RegisterPOI(this);
	}
}
```