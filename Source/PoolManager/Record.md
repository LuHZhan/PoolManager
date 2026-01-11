```
UPoolManagerSubsystem::TakeFromPool ->

   UPoolManagerSubsystem::TakeFromPoolOrNull ->
   
   -> ObjectData == nullptr 
   
      UPoolManagerSubsystem::CreateNewObjectInPool_Implementation ->
   
      UPoolManagerSubsystem::FindPoolOrAdd ->
      
         -> Pool == nullptr 
      
         UPoolManagerSubsystem::FindPoolFactoryChecked ->
      
            AllFactoriesInternal.Find(CurrentClass) ->

            ...
            
         ...
         
      ...
      
      UPoolManagerSubsystem::SetObjectStateInPool ->
      
   [end]
   
[end]
   
```