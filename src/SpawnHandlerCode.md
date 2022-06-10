```csharp
[EntityEditorProps(category: "GameScripted/Systems", description: "Entity that takes care of managing AI spawning.", color: "0 0 255 255")]
class SCR_Badger_SpawnHandlerClass: GenericEntityClass
{
};

class SCR_Badger_SpawnHandler : GenericEntity
{
	// D E F E N D I N G   V A R I A B L E S 
	[Attribute("", UIWidgets.EditBox, desc: "Name of defending faction", category: "Badger: Defending")]
	private FactionKey m_DefendingFactionKey;
	
	[Attribute("0", UIWidgets.Slider, desc: "Amount of defending unit groups", category: "Badger: Defending", params: "1, 100, 1")]
	private int m_DefendingSideGroupCount;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Prefab to be spawned",  params: "et", category: "Badger: Defending")]
	private ref array<ResourceName> m_DefendingFactionGroupPrefabs;
	
	[Attribute("", UIWidgets.EditBox, category: "Badger: Defending")]
	private ref array<string> m_DefendingSpawnLocationNames;
	
	// A T T A C K I N G    V A R I A B L E S
	[Attribute("", UIWidgets.EditBox, desc: "Name of attacking faction", category: "Badger: Attacking")]
	private FactionKey m_AttackingFactionKey;
	
	[Attribute("0", UIWidgets.Slider, desc: "Amount of attacking unit groups", category: "Badger: Attacking", params: "1, 100, 1")]
	private int m_AttackingSideGroupCount;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Prefab to be spawned",  params: "et", category: "Badger: Attacking")]
	private ref array<ResourceName> m_AttackingFactionGroupPrefabs;
	
	[Attribute("", UIWidgets.EditBox, category: "Badger: Attacking")]
	private ref array<string> m_AttackingSpawnLocationNames;	
	
	[Attribute("", UIWidgets.Slider, category: "Badger: General", params: "0 500, 1")]
	private int m_GeneralSpawnRadius;
	
	[Attribute("3", UIWidgets.Slider, desc: "Number of groups that spawn per interval", category: "Badger: Spawn Rate", params: "1, 20, 1")]
	private int m_MinimumSpawnPerInterval;
	
	[Attribute("10", UIWidgets.Slider, desc: "Number of groups that spawn per interval", category: "Badger: Spawn Rate", params: "1, 20, 1")]
	private int m_MaxiumSpawnPerInterval;	
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Pick Waypoint prefab for attackers", category: "Badger: Waypoint Prefabs")]
	private ResourceName m_AttackWaypointPrefab;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Pick Waypoint prefab for defenders", category: "Badger: Waypoint Prefabs")]
	private ResourceName m_DefendWaypointPrefab;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Pick Waypoint prefab for cycling waypoints", category: "Badger: Waypoint Prefabs")]
	private ResourceName m_CycleWaypoint;
	
	[Attribute("1", UIWidgets.CheckBox, desc: "Should building prefabs be placed at start?", category: "Badger: Structures")]	
	private bool m_PlacePrefabsOnStart;	
	
	[Attribute("500", UIWidgets.Slider, desc: "Distance from Spawn Location where controlled faction's buildings appear", category: "Badger: Structures", params: "100, 1000, 1")]
	private int m_structureQueryDistance;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Road Site Prefabs of controlled areas", category: "Badger: Attack - Sites", params: "et")]
	private ref array<ResourceName> m_attackingRoadSitePrefabs;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Small Site Prefabs of controlled areas", category: "Badger: Attack - Sites", params: "et")]
	private ref array<ResourceName> m_attackingSmallSitePrefabs;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Medium Site Prefabs of controlled areas", category: "Badger: Attack - Sites", params: "et")]
	private ref array<ResourceName> m_attackingMediumSitePrefabs;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Large Site Prefabs of controlled areas", category: "Badger: Attack - Sites", params: "et")]
	private ref array<ResourceName> m_attackingLargeSitePrefabs;	
		
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Road Site Prefabs of controlled areas", category: "Badger: Defending - Sites", params: "et")]
	private ref array<ResourceName> m_defendingRoadSitePrefabs;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Small Site Prefabs of controlled areas", category: "Badger: Defending - Sites", params: "et")]
	private ref array<ResourceName> m_defendingSmallSitePrefabs;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Medium Site Prefabs of controlled areas", category: "Badger: Defending - Sites", params: "et")]
	private ref array<ResourceName> m_defendingMediumSitePrefabs;
	
	[Attribute("", UIWidgets.ResourcePickerThumbnail, desc: "Large Site Prefabs of controlled areas", category: "Badger: Defending - Sites", params: "et")]
	private ref array<ResourceName> m_defendingLargeSitePrefabs;
		
	// Referemce to our game world since we'll need it for spawning stuff	
	BaseWorld world;
	
	// The number of AI Groups per faction that currently exist in-game
	private int m_CurrentDefendingGroupCount = 0;
	private int m_CurrentAttackingGroupCount = 0;		
	
	// If we're calculating the number of AI, we're going to cease spawning
	private bool m_IsCalculatingSides = false;
	private bool m_IsLoadingLocations = true;
	
	private ref array<IEntity> defendingSpawnLocations = {};
	private ref array<IEntity> attackingSpawnLocations = {};
	
	private ref array<SCR_SiteSlotEntity> m_aSlotEntities = new array<SCR_SiteSlotEntity>();	
	
	ref RandomGenerator random;
	FactionManager factionManager;
	
	void SCR_Badger_SpawnHandler(IEntitySource src, IEntity parent)
	{
		SetEventMask(EntityEvent.INIT);
		Activate();
	}
	
	protected override void EOnInit(IEntity owner)
	{		
		if(!GetGame().InPlayMode())
			return;
		
		world = GetGame().GetWorld();
		factionManager = GetGame().GetFactionManager();
		random = new RandomGenerator();
		
		Print("Loading locations for Badger Spawn Handler", LogLevel.WARNING);
		LoadLocations();

		Print("Initializing Sites for Badger Spawn Handler", LogLevel.WARNING);
		InitializeSites();
				
		Print("Starting spawn loop for Badger Spawn Handler", LogLevel.WARNING);
		SpawnLoop();
	}
	
	private bool queryingAttackers = false;
	private ref map<SCR_ESlotTypesEnum, ref array<SCR_SiteSlotEntity>> defendingSlots = new map<SCR_ESlotTypesEnum, ref array<SCR_SiteSlotEntity>>;
	private ref map<SCR_ESlotTypesEnum, ref array<SCR_SiteSlotEntity>> attackingSlots = new map<SCR_ESlotTypesEnum, ref array<SCR_SiteSlotEntity>>;
	
	bool QueryEntitiesCallback(IEntity e)
	{
		SCR_SiteSlotEntity slot = SCR_SiteSlotEntity.Cast(e);		
		
		if(slot)
		{
			m_aSlotEntities.Insert(slot);
			
			string prefabName = slot.GetPrefabData().GetPrefabName();			
			Print(string.Format("Found Slot: %1", prefabName), LogLevel.WARNING);
			
			if(prefabName.EndsWith("E_SlotFlatSmall.et"))
			{
				if(queryingAttackers)
					AddAttackingSlot(slot, SCR_ESlotTypesEnum.FlatSmall);
				else
					AddDefendingSlot(slot, SCR_ESlotTypesEnum.FlatSmall);
			}
			else if(prefabName.EndsWith("E_SlotFlatMedium.et"))
			{
				if(queryingAttackers)
					AddAttackingSlot(slot, SCR_ESlotTypesEnum.FlatMedium);
				else
					AddDefendingSlot(slot, SCR_ESlotTypesEnum.FlatMedium);
			}
			else if(prefabName.EndsWith("E_SlotFlatLarge.et"))
			{
				if(queryingAttackers)
					AddAttackingSlot(slot, SCR_ESlotTypesEnum.FlatLarge);
				else
					AddDefendingSlot(slot, SCR_ESlotTypesEnum.FlatLarge);
			}
			else if(prefabName.EndsWith("E_SlotRoadSmall.et"))
			{
				if(queryingAttackers)
					AddAttackingSlot(slot, SCR_ESlotTypesEnum.CheckpointSmall);
				else
					AddDefendingSlot(slot, SCR_ESlotTypesEnum.CheckpointSmall);
			}
			else if(prefabName.EndsWith("E_SlotRoadMedium.et"))
			{
				if(queryingAttackers)
					AddAttackingSlot(slot, SCR_ESlotTypesEnum.CheckpointMedium);
				else
					AddDefendingSlot(slot, SCR_ESlotTypesEnum.CheckpointMedium);
			}
			else if(prefabName.EndsWith("E_SlotRoadLarge.et"))
			{
				if(queryingAttackers)
					AddAttackingSlot(slot, SCR_ESlotTypesEnum.CheckpointLarge);
				else
					AddDefendingSlot(slot, SCR_ESlotTypesEnum.CheckpointLarge);
			}			
		}				
		
		return true;
	}
	
	void AddDefendingSlot(SCR_SiteSlotEntity slot, SCR_ESlotTypesEnum slotType)
	{
		if(!defendingSlots.Contains(slotType))
			defendingSlots.Insert(slotType, new array<SCR_SiteSlotEntity>());
		
		defendingSlots.Get(slotType).Insert(slot);
	}
	
	void AddAttackingSlot(SCR_SiteSlotEntity slot, SCR_ESlotTypesEnum slotType)
	{
		if(!attackingSlots.Contains(slotType))
			attackingSlots.Insert(slotType, new array<SCR_SiteSlotEntity>());
		attackingSlots.Get(slotType).Insert(slot);
	}
	
	void InitializeSites()
	{
		if(!m_PlacePrefabsOnStart)
			return;
		
		// Reset, just in case.
		m_aSlotEntities.Clear();
		
		// We need to query each spawn location for the appropriate sites
		foreach(IEntity location : defendingSpawnLocations)
		{
			vector center = location.GetOrigin();			
			world.QueryEntitiesBySphere(center, m_structureQueryDistance, QueryEntitiesCallback, null, EQueryEntitiesFlags.ALL);
		}
			
		queryingAttackers = true;
		
		foreach(IEntity location : attackingSpawnLocations)
		{
			vector center = location.GetOrigin();			
			world.QueryEntitiesBySphere(center, m_structureQueryDistance, QueryEntitiesCallback, null, EQueryEntitiesFlags.ALL);
		}
		
		Print(string.Format("Attacking Sites: %1 | Defending Sites: %2", attackingSlots.Count(), defendingSlots.Count()), LogLevel.WARNING);
		CreateSites();
	}
	
	void CreateSites()
	{
		foreach(SCR_ESlotTypesEnum slotType, ref array<SCR_SiteSlotEntity> slots: defendingSlots)
		{
			foreach(SCR_SiteSlotEntity slot : slots)
			{
				switch(slotType)
				{
					case SCR_ESlotTypesEnum.FlatSmall:
					{
						SpawnSite(m_defendingSmallSitePrefabs, slot);						
						break;
					}
					case SCR_ESlotTypesEnum.FlatMedium:
					{
						SpawnSite(m_defendingMediumSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.FlatLarge:
					{
						SpawnSite(m_defendingLargeSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.CheckpointSmall:
					{
						SpawnSite(m_defendingRoadSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.CheckpointMedium:
					{
						SpawnSite(m_defendingRoadSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.CheckpointLarge:
					{
						SpawnSite(m_defendingRoadSitePrefabs, slot);
						break;
					}
				}
			}
		}
		
		foreach(SCR_ESlotTypesEnum slotType, ref array<SCR_SiteSlotEntity> slots: attackingSlots)
		{
			foreach(SCR_SiteSlotEntity slot : slots)
			{
				switch(slotType)
				{
					case SCR_ESlotTypesEnum.FlatSmall:
					{
						SpawnSite(m_attackingSmallSitePrefabs, slot);						
						break;
					}
					case SCR_ESlotTypesEnum.FlatMedium:
					{
						SpawnSite(m_attackingMediumSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.FlatLarge:
					{
						SpawnSite(m_attackingLargeSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.CheckpointSmall:
					{
						SpawnSite(m_attackingRoadSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.CheckpointMedium:
					{
						SpawnSite(m_attackingRoadSitePrefabs, slot);
						break;
					}
					case SCR_ESlotTypesEnum.CheckpointLarge:
					{
						SpawnSite(m_attackingRoadSitePrefabs, slot);
						break;
					}
				}
			}
		}
	}
	
	void SpawnSite(array<ResourceName> collection, SCR_SiteSlotEntity slot)
	{
		if(slot == null)
			return;
		
		if(slot.IsOccupied())
		{
			Print(string.Format("Site %1 already has stuff there, skipping",slot.GetName()), LogLevel.WARNING);
			return;
		}
		
		ResourceName randomResource = collection.GetRandomElement();
		Resource resource = Resource.Load(randomResource);
		Print(string.Format("Spawning Site: %1", randomResource.GetPath()));
		slot.SpawnEntityInSlot(resource);					
	}
	
	void LoadLocations()
	{		
		foreach(string name : m_DefendingSpawnLocationNames)
		{
			IEntity location = world.FindEntityByName(name);
			
			if(location)
			{
				defendingSpawnLocations.Insert(location);
			}
		}
		
		foreach(string name : m_AttackingSpawnLocationNames)
		{
			IEntity location = world.FindEntityByName(name);
			
			if(location)
			{
				attackingSpawnLocations.Insert(location);
			}
		}
		
		m_IsLoadingLocations = false;
	}
	
	void SpawnLoop()
	{
		if(!GetGame())
			Print("[Badger Spawn Handler] Unable to find game instance", LogLevel.ERROR);
		
		// Load our spawn locations
		if(defendingSpawnLocations.IsEmpty() || attackingSpawnLocations.IsEmpty())
		{
			LoadLocations();
		}
		
		if(m_CurrentDefendingGroupCount < m_DefendingSideGroupCount)
		{
			// Spawn defending stuff
			int count = RandomCountFor(false);
			for(int i = 0; i < count; i++)
			{
				// Select a random prefab
				ResourceName prefabName = m_DefendingFactionGroupPrefabs.GetRandomElement();
				IEntity location = defendingSpawnLocations.GetRandomElement();
				Spawn(prefabName, location, false);
			}			
		}
		
		if(m_CurrentAttackingGroupCount < m_AttackingSideGroupCount)
		{
			int count = RandomCountFor(true);
			for(int i = 0; i < count; i++)
			{
				// select random prefab
				ResourceName prefab = m_AttackingFactionGroupPrefabs.GetRandomElement();
				IEntity location = attackingSpawnLocations.GetRandomElement();
				Spawn(prefab, location, true);
			}
		}
		
		// Recursion		
		CountSides();
		GetGame().GetCallqueue().CallLater(SpawnLoop,  120 * 1000, false);
	}
	
	void CountSides()
	{
		m_IsCalculatingSides = true;
		
		ref private array<AIAgent> m_entities = {};		
		AIWorld aiWorld = GetGame().GetAIWorld();
		aiWorld.GetAIAgents(m_entities);
		
		m_CurrentDefendingGroupCount = 0;
		m_CurrentAttackingGroupCount = 0;
		
		Print(string.Format("Filtering units from pool of %1", m_entities.GetRefCount()), LogLevel.WARNING);
		
		foreach(AIAgent agent : m_entities)
		{
			SCR_AIGroup chimera = SCR_AIGroup.Cast(agent);
			
			if(!chimera)
			{
				Print(string.Format("Was not a SCR_AIGroup but a %1", agent.ClassName()), LogLevel.WARNING);
				continue;
			}
			
			if(chimera.GetFaction().GetFactionKey() == m_AttackingFactionKey)
				m_CurrentAttackingGroupCount += 1;
			else if(chimera.GetFaction().GetFactionKey() == m_AttackingFactionKey)
				m_CurrentDefendingGroupCount += 1;
		}
		
		Print(string.Format("Defending: %1 Attacking: %2", m_CurrentDefendingGroupCount, m_CurrentAttackingGroupCount), LogLevel.WARNING);				
		m_IsCalculatingSides = false;	
	}
	
	vector RandomPositionAround(IEntity point, int radius)
	{
		vector mat[4];
		point.GetWorldTransform(mat);
		
		vector position = mat[3];
		position = random.GenerateRandomPointInRadius(0, radius, position);
		
		return position;
	}
	
	bool Spawn(ResourceName prefab, IEntity spawnPoint, bool isAttacking=true)
	{	
		// We need the world and prefab
		// We also need spawn locations
		if(!world|| prefab.IsEmpty() || m_IsCalculatingSides || m_IsLoadingLocations)
			return false;
		
		Resource resource = Resource.Load(prefab);
		EntitySpawnParams params();	
		
		// Position of spawn point
		vector mat[4];
		spawnPoint.GetWorldTransform(mat);
		
		// We need to generate a random position in area
		vector position = mat[3];
		position = random.GenerateRandomPointInRadius(0, m_GeneralSpawnRadius, position);
		
		// Ensure the position is snapped to ground
		position[1] = spawnPoint.GetWorld().GetSurfaceY(position[0], position[2]);
		mat[3] = position;
		
		// Update spawn params
		params.TransformMode = ETransformMode.WORLD;
		params.Transform = mat;
		
		SCR_AIGroup newEntity = SCR_AIGroup.Cast(GetGame().SpawnEntityPrefab(resource, world, params));
		
		if(!newEntity)
			return false;
		
		newEntity.SetFlags(EntityFlags.VISIBLE, true);
		OnSpawn(newEntity);
				
		SetWaypointFor(newEntity, isAttacking);
		
		return true;
	}
	
	int RandomCountFor(bool attackingTeam = true)
	{
		int minimum = 1;
		int maximum = 1;
		int difference = 0;
		
		if(attackingTeam)
			difference = m_AttackingSideGroupCount - m_CurrentAttackingGroupCount;			
		else
			difference = m_DefendingSideGroupCount - m_CurrentDefendingGroupCount;
					
		// Difference will mean we are over the limit thus shouldn't spawn anything more
		if(difference < 0) return 0;
		maximum = Math.Ceil(Math.Max(1, difference));
		return Math.RandomInt(minimum, Math.Max(1, maximum));
	}
	
	AIWaypoint CreateWaypoint(ResourceName waypointPrefab)
	{
		Resource resource = Resource.Load(waypointPrefab);
		
		if(!resource)
			return null;
		
		AIWaypoint wp = AIWaypoint.Cast(GetGame().SpawnEntityPrefab(resource));
		
		if(!wp)
			return null;
		
		return wp;	
	}
	
	void CreatePatrolPathFor(SCR_AIGroup group)
	{
		int waypointCount = Math.RandomInt(2, 6);
		
		for(int i = 0; i < waypointCount; i++)
		{
			IEntity point = defendingSpawnLocations.GetRandomElement();
			AIWaypoint waypoint = CreateWaypoint(m_DefendWaypointPrefab);
			
			if(!waypoint)
				break;
			
			vector position = RandomPositionAround(point, 50);			
			waypoint.SetOrigin(position);
			group.AddWaypoint(waypoint);
		}
		
		// This will make the group repeat their patrol path
		AIWaypoint cycle = CreateWaypoint(m_CycleWaypoint);
		cycle.SetOrigin(group.GetOrigin());
		group.AddWaypoint(cycle);
	}
	
	void SetWaypointFor(SCR_AIGroup group, bool isAttacking=true)
	{
		IEntity randomLocation = defendingSpawnLocations.GetRandomElement();
		
		AIWaypoint waypoint;
		
		if(isAttacking)
			waypoint = CreateWaypoint(m_AttackWaypointPrefab);
		else
		{
			CreatePatrolPathFor(group);
			return;
		}
		
		if(!waypoint)
		{
			Print("Error creating waypoint for attacking team", LogLevel.WARNING);
			return;
		}
		
		waypoint.SetOrigin(randomLocation.GetOrigin());				
		group.AddWaypoint(waypoint);
	}
	
	protected void OnSpawn(IEntity newEntity)
	{
	
	}
};

```