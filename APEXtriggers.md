trigger ClosedOpportunityTrigger on Opportunity (after insert, after update) {
List<Task> taskList = new List<Task>();
    for(Opportunity i: Trigger.new){
        if(i.StageName == 'Closed Won'){
            Task newTask = new Task(
            Subject = 'Follow Up Test Task',
            WhatId = i.Id
            );
                        taskList.add(newTask);

        }
      
    }
    if(taskList.size() > 0){
        insert taskList;
    }
}
-------
trigger AccountAddressTrigger on Account (before insert, before update) {
    for(Account i : Trigger.new){
        if (i.Match_Billing_Address__c == true && i.BillingPostalCode!= null ){
			i.ShippingPostalCode=i.BillingPostalCode;
                }
    }
    
}
-------
trigger AccountDeletion on Account (before delete) {
  // Prevent the deletion of accounts if they have related opportunities.
  for(Account a : [SELECT Id FROM Account
    WHERE Id IN (SELECT AccountId FROM Opportunity) AND
    Id IN :Trigger.old]) {
    Trigger.oldMap.get(a.Id).addError('Cannot delete account with related opportunities.');
  }
}
-------
trigger AddRelatedRecord on Account(after insert, after update) {
    List<Opportunity> oppList = new List<Opportunity>();
    // Add an opportunity for each account if it doesn't already have one.
    // Iterate over accounts that are in this trigger but that don't have opportunities.
    List<Account> toProcess = null;
    switch on Trigger.operationType {
        when AFTER_INSERT {
        // All inserted Accounts will need the Opportunity, so there is no need to perform the query
            toProcess = Trigger.New;
        }
        when AFTER_UPDATE {
            toProcess = [SELECT Id,Name FROM Account
                         WHERE Id IN :Trigger.New AND
                         Id NOT IN (SELECT AccountId FROM Opportunity WHERE AccountId in :Trigger.New)];
        }
    }
    for (Account a : toProcess) {
        // Add a default opportunity for this account
        oppList.add(new Opportunity(Name=a.Name + ' Opportunity',
                                    StageName='Prospecting',
                                    CloseDate=System.today().addMonths(1),
                                    AccountId=a.Id));
    }
    if (oppList.size() > 0) {
        insert oppList;
    }
}
-------