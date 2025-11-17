
UDamageType
> A DamageType is intended to define and describe a particular form of damage and to provide an avenue for customizing responses to damage from various sources.  
> For example, a game could make a DamageType_Fire set it up to ignite the damaged actor.  DamageTypes are never instanced and should be treated as immutable data holders with static code functionality.  They should never be stateful. 

DamageType 的作用是定义和描述特定类型的伤害，并提供定制不同伤害来源响应的途径。例如，游戏可以设置 `DamageType_Fire`，使其能够点燃受损的Actor。DamageType 永远不会被实例化，应将其视为不可变的数据容器，具备静态代码功能，且不应包含任何状态

DamageType的使用方式是只使用它的默认值，从CO0中拿值。所有定义Damage的方式是继承`UDamageType`，然后为其设置默认值。运行时修改不会被使用

### PointDamage
参数：
	`Damage -> float`
	`HitInfo -> FHitResult`
	

### RadialDamage
参数：
	`Params -> FRadialDamageParams`
	`Origin -> FVector`
	`ComponentHits -> FHitResult`
```cpp
struct FRadialDamageParams
{
	/** Max damage done */  
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=RadialDamageParams)  
	float BaseDamage;  
	  
	/** Damage will not fall below this if within range */  
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=RadialDamageParams)  
	float MinimumDamage;  
	  
	/** Within InnerRadius, do max damage */  
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=RadialDamageParams)  
	float InnerRadius;  
	      
	/** Outside OuterRadius, do no damage */  
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=RadialDamageParams)  
	float OuterRadius;  
	      
	/** Describes amount of exponential damage falloff */  
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=RadialDamageParams)  
	float DamageFalloff;
}
```

