# Black Tower Assignment



The project is built upon base Lyra in Unreal Engine 5.4. Since Lyra is a highly modular template, usually you don't need to work a lot on the C++ apart from just deriving your own classes. Thus the project is built upon using mostly Lyra C++ classes apart from some of my custom C++ classes.
Also, I did not find a good showcase of C++ use in my base classes apart from just setting up the Inputs and Ability System. But I still found a good use-case that involves modifying some of the Lyra code. I have described it at the end of this document. 

## Overview of the Project:

There is a demo video present in the zip file. But here is the high level overview of what to look for in the project

- I have created a separate GameFeature Plugin that contains all my code. This was the suggested way by Lyra pros from Unreal Engine streams.

  

- Regarding the C++ I have created the following classes 

- - `UBlackTowerCharacterBase` is the base class for the player character derived from `ULyraCharacter`

  - `UBlackTowerCharacterWithAbilities` is a child of `UBlackTowerCharacterBase` but with the AbilitySystemComponent and a PawnData. The purpose of creating this class instead of using `ULyraCharacterWithAbilities` is to be used as an enemy base class to able to use the "Infest" ability described later in this doc.

  - `UBlackTowerHeroComponent` is a child of `ULyraHeroComponent` and involves some necessary tweaks for the "Infest" ability

  - `UAbilityTask_OnTick`  an ability task that is used in the `GA_Infest`

    

- In the Content directory, there are a few things as well

- - Firstly, as said above, all the content related to this assignment will be in the content directory of the GameFeature `BlackTowerCore`
  - `Experiences` folder contains the only experience definition for this GFP
  - `Game` folder contains all the Blueprints for the character, enemies, related data assets, as well as the Gameplay Abilities and some other BPs as well.
  - `Weapons` folder contains all the assets related to the only weapon of this project `The Gravity Gun`

- `Gravity Gun` gives the ability to pick up and move objects as well as enemies around. It is spawned in the world using a `LyraWeaponSpawner` after activating a specific trigger box

- `Infest` is an ability that allows the character to possess other enemies in the game and be able to control them. `GA_Infest` and `GA_ReturnToCharacter` are abilities used to possess an enemy and return to the main character respectively. 

- The `Infest` ability was the most time consuming as the Lyra doesn't readily allows you to possess any pawn. If you leave the character and possess another pawn, it will not have the same ability system reference, nor it will have the correct initializations and input. So with some help from forums I was able to figure out what the issue with Lyra is and had to do some destructive changes to Lyra base classes to fix them.

- Below is a list of all the changes in C++, some related to the above issue, and some other changes as well.

  

### MODIFICATIONS TO `ULyraCharacter`

Apparently, the pawn component is not exposed by default (which is kind of weird), but I needed it to do some reinitializations mentioned later, so I had to create a Getter.

```cpp

	// Getter for Pawn Component
	inline TObjectPtr<ULyraPawnExtensionComponent> GetPawnComponent() const { return PawnExtComponent; }

```



## Supporting targetable abilities

For Infest ability, I wanted it to be a targetable ability where user enters the targeting mode, selects a target and then clicks to possess it. For that purpose I had to create two new inputs and bind them natively to the ability system's input actions. I think it's just a basic need for most projects and should be there by default just like Move and Look inputs.

##### `LyraGameplayTags.h`

```cpp
LYRAGAME_API    UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Confirm);
LYRAGAME_API    UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Cancel);
```

##### `UBlackTowerHeroComponent::InitializePlayerInput`

###### Had to add `LYRAGAME_API`macro to `ULyraInputComponent` and `ULyraInputConfig` Classes  also to `ULyraAbilitySet`, `ULyraHealthSet` and `ULyraCombatSet`

```cpp
if (const ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
{
    if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
    {
       if (const ULyraInputConfig* InputConfig = PawnData->InputConfig)
       {
          ULyraInputComponent* LyraIC = Cast<ULyraInputComponent>(PlayerInputComponent);
          if (ensureMsgf(LyraIC, TEXT("Unexpected Input Component class! The Gameplay Abilities will not be bound to their inputs. Change the input component to ULyraInputComponent or a subclass of it.")))
          {
             if (ULyraAbilitySystemComponent* LyraASC = PawnExtComp->GetLyraAbilitySystemComponent())
             {
                LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Confirm, ETriggerEvent::Triggered, LyraASC, &UAbilitySystemComponent::LocalInputConfirm, /*bLogIfNotFound=*/ false);
                LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Cancel, ETriggerEvent::Triggered, LyraASC, &UAbilitySystemComponent::LocalInputCancel, /*bLogIfNotFound=*/ false);
             }
          }
       }
    }
}
```





## Making Possession Work

#### Explicitly register the Hero component and call InitializePlayerInput

```cpp
void ABlackTowerCharacterBase::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    // Reinit Player input on the Hero Component
    if (PlayerInputComponent != nullptr)
    {
       if (UBlackTowerHeroComponent* BTHeroComp = FindComponentByClass<UBlackTowerHeroComponent>())
       {
          BTHeroComp->RegisterComponent();
          BTHeroComp->InitializePlayerInput(PlayerInputComponent);
       }
    }  
}
```

#### Comment out this code in `ULyraHeroComponent::HandleChangeInitState()` as we are now Initializing input in the character. This is to prevent errors due to multiple initializations

```cpp
#pragma region BLACK TOWER MODIFICATION
		/*
		if (ALyraPlayerController* LyraPC = GetController<ALyraPlayerController>())
		{
			if (Pawn->InputComponent != nullptr)
			{
				InitializePlayerInput(Pawn->InputComponent);
			}
		}
		*/
#pragma endregion
```

#### Unregistering Hero Comp and Ability System on `UnPossessed`

```cpp
void ASCLCharacterBase::UnPossessed()
{
    Super::UnPossessed();
    
    if (USCLPawnComponent* SCL_PawnComp = FindComponentByClass<USCLPawnComponent>())
    {
        SCL_PawnComp->UnregisterComponent();
    }

    // TODO: Do it in a better way
    OnAbilitySystemUninitialized();
}
```

#### In the `OnRegister()` Function of the `UBlackTowerHeroComponent` set this bool to false. This causes some ensures to trigger.

```cpp
void USCLPawnComponent::OnRegister()
{
	Super::OnRegister();

	// Set this to false to prevent triggering ensure on re-init player input
	bReadyToBindInputs = false;
}
```

##### Even after all this, when repossessing the pawn, it was left without an AbilitySystemComponent or an older one. From the forums I learned the the PawnExtensionComponent is caching the AbilitySystemComponent but when the controller changes it's not updating the reference. So had to replace the entire `ULyraPawnExtensionComponent::HandleControllerChanged()` funciton with this code

```cpp
void ULyraPawnExtensionComponent::HandleControllerChanged()
{
#pragma region BLACK TOWER MODIFICATION
	
	ULyraAbilitySystemComponent* NewASC = nullptr;
	if (APawn* Pawn = GetPawnChecked<APawn>()) {
		if (ALyraPlayerState* LyraPS = Cast<ALyraPlayerState>(Pawn->GetPlayerState())) {
			NewASC = LyraPS->GetLyraAbilitySystemComponent();
		}
	}

	if (NewASC == nullptr) {
		UninitializeAbilitySystem();
	}
	else if (NewASC != AbilitySystemComponent) {
		InitializeAbilitySystem(NewASC, GetOwningActor());
	}
	else {
		AbilitySystemComponent->RefreshAbilityActorInfo();
	}

	CheckDefaultInitialization();

#pragma endregion
}
```

#### Now that we have a working ability system and correct pawn initialization, next was to get the Input working again, Lyra uses the `PawnData` data asset that contains the `AbilitySet` and the ` InputConfig`. I had to give the pawn data to the enemy class as well and then correctly initialize it when the player possess the enemy. For that I added some extension variables and functions in the `UBlackTowerCharacterWithAbilities`. Not attaching the code here as that entire class is somewhat related to this.

