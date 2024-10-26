
In DellPkgs\DellPublicPkgs\DellPlatformFeaturePkgs\DellTcg\DellTcg2PlatformDxe\DellTcg2PlatformDxe.c
	1. In DellHandleTpm20Setup(),
	   CreateEventEx with PwMgrEventPwAuthedGuid, 
	   which will call ChassisIntrusionHandler() during BDS, 
	   if AdminPw incorrect && PID_TPM_CLEAR is on.
	2. And then DellTcg2CommandClear()
	     - Tpm2ClearControl()
	     - Tpm2Clear()
	3. 
	   
In Edk2\edk2-LNL\SecurityPkg\Library\Tpm2CommandLib\Tpm2Hierarchy.c
1. Tpm2ClearControl()
     Tpm2SubmitCommand()
2. .