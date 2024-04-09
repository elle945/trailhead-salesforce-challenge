## TEST APEX TRIGGERS
```

trigger RestrictContactByName on Contact (before insert, before update) {
  //check contacts prior to insert or update for invalid data
  for(Contact c : Trigger.New) {
    if(c.LastName == 'INVALIDNAME') {
      //invalidname is invalid
      c.AddError('The Last Name "'+c.LastName+'" is not allowed for DML');
    }
  }
}
´´´
```

@isTest
public class TestRestrictContactByName {
static testMethod void  Test() 
    {
    
        List<Contact> listContact= new List<Contact>();
        Contact c1 = new Contact(FirstName='Raam', LastName='Leela' , email='ramleela@test.com');
        Contact c2 = new Contact(FirstName='gatsby', LastName = 'INVALIDNAME',email='gatsby@test.com');
        listContact.add(c1);
        listContact.add(c2);
        
        Test.startTest();
            try
            {
                insert listContact;
            }
            catch(Exception ee)
            {
            }
        
        Test.stopTest(); 
        
    }
}
´´´


## TEST ACCOUNT DELETION
```

trigger AccountDeletion on Account (before delete) {
  // Prevent the deletion of accounts if they have related opportunities.
  for(Account a : [SELECT Id FROM Account
    WHERE Id IN (SELECT AccountId FROM Opportunity) AND
    Id IN :Trigger.old]) {
    Trigger.oldMap.get(a.Id).addError('Cannot delete account with related opportunities.');
  }
}
´´´
```

@isTest
private class TestAccountDeletion {
  @isTest
  static void TestDeleteAccountWithOneOpportunity() {
    // Test data setup
    // Create one account with one opportunity by calling a utility method
    Account[] accts = TestDataFactory.createAccountsWithOpps(1,1);
    // Perform test
    Test.startTest();
      Database.DeleteResult result = Database.delete(accts[0], false);
    Test.stopTest();
    // Verify that the deletion should have been stopped by the trigger,
    // so check that we got back an error.
    System.assert(!result.isSuccess());
    System.assert(result.getErrors().size() > 0);
    System.assertEquals('Cannot delete account with related opportunities.',
      result.getErrors()[0].getMessage());
  }
  @isTest
  static void TestDeleteAccountWithNoOpportunities() {
    // Test data setup
    // Create one account with no opportunities by calling a utility method
    Account[] accts = TestDataFactory.createAccountsWithOpps(1,0);
    // Perform test
    Test.startTest();
      Database.DeleteResult result = Database.delete(accts[0], false);
    Test.stopTest();
    // Verify that the deletion was successful
    System.assert(result.isSuccess());
  }
  @isTest
  static void TestDeleteBulkAccountsWithOneOpportunity() {
    // Test data setup
    // Create accounts with one opportunity each by calling a utility method
    Account[] accts = TestDataFactory.createAccountsWithOpps(200,1);
    // Perform test
    Test.startTest();
      Database.DeleteResult[] results = Database.delete(accts, false);
    Test.stopTest();
    // Verify for each record.
    // In this case the deletion should have been stopped by the trigger,
    // so check that we got back an error.
    for(Database.DeleteResult dr : results) {
      System.assert(!dr.isSuccess());
      System.assert(dr.getErrors().size() > 0);
      System.assertEquals('Cannot delete account with related opportunities.',
        dr.getErrors()[0].getMessage());
    }
  }
  @isTest
  static void TestDeleteBulkAccountsWithNoOpportunities() {
    // Test data setup
    // Create accounts with no opportunities by calling a utility method
    Account[] accts = TestDataFactory.createAccountsWithOpps(200,0);
    // Perform test
    Test.startTest();
      Database.DeleteResult[] results = Database.delete(accts, false);
    Test.stopTest();
    // For each record, verify that the deletion was successful
    for(Database.DeleteResult dr : results) {
      System.assert(dr.isSuccess());
    }
  }
}
´´´

## VERIFY DATE 

public class VerifyDate {
  //method to handle potential checks against two dates
  public static Date CheckDates(Date date1, Date date2) {
    //if date2 is within the next 30 days of date1, use date2.  Otherwise use the end of the month
    if(DateWithin30Days(date1,date2)) {
      return date2;
    } else {
      return SetEndOfMonthDate(date1);
    }
  }
  //method to check if date2 is within the next 30 days of date1
  private static Boolean DateWithin30Days(Date date1, Date date2) {
    //check for date2 being in the past
    if( date2 < date1) { return false; }
    //check that date2 is within (>=) 30 days of date1
    Date date30Days = date1.addDays(30); //create a date 30 days away from date1
    if( date2 >= date30Days ) { return false; }
    else { return true; }
  }
  //method to return the end of the month of a given date
  private static Date SetEndOfMonthDate(Date date1) {
    Integer totalDays = Date.daysInMonth(date1.year(), date1.month());
    Date lastDay = Date.newInstance(date1.year(), date1.month(), totalDays);
    return lastDay;
  }
}


```
@isTest
public class TestVerifyDate {
    @isTest
    static void testDaysBetweenDates(){
        Date d1 = Date.newInstance(2024, 01, 01);
        Date d2 = Date.newInstance(2024, 02, 02);
        Date olddate = Date.newInstance(2023, 10, 01);
        Date date1 = VerifyDate.CheckDates(d1, d2);
        Date oDate = VerifyDate.CheckDates(d1, olddate);
    }
    @isTest
    static void testDate2BeforeDate1(){
        Date date1 = Date.newInstance(2024, 01, 10);
        Date date2 = Date.newInstance(2024, 01, 01);
        Date dateResult = VerifyDate.CheckDates(date1, date2);
		Date expectedEndOfMonth = Date.newInstance(2024, 1, 31);
            System.assertEquals(expectedEndOfMonth, dateResult);
    }
        @isTest
    static void testDate2NotWithin30Days() {
        Date d1 = Date.newInstance(2024, 1, 1);
        Date d2 = Date.newInstance(2024, 2, 5); // date2 is more than 30 days away from date1
        Date result = VerifyDate.CheckDates(d1, d2);
        // Don't need to call SetEndOfMonthDate directly, just assert the expected result
        Date expectedEndOfMonth = Date.newInstance(2024, 1, 31); // Manually calculate expected result
        System.assertEquals(expectedEndOfMonth, result, 'Should return end of month if date2 is not within 30 days of date1.');
    }

    @isTest
    static void testDate2Within30Days() {
        Date d1 = Date.newInstance(2024, 1, 1);
        Date d2 = Date.newInstance(2024, 1, 29); // date2 is within 30 days of date1
        Date result = VerifyDate.CheckDates(d1, d2);
        System.assertEquals(d2, result, 'Should return date2 if it is within 30 days of date1.');
    }
}
´´´
```

public class TemperatureConverter {
  // Takes a Fahrenheit temperature and returns the Celsius equivalent.
  public static Decimal FahrenheitToCelsius(Decimal fh) {
    Decimal cs = (fh - 32) * 5/9;
    return cs.setScale(2);
  }
}
´´´
```

@isTest
private class TemperatureConverterTest {
  @isTest static void testWarmTemp() {
    Decimal celsius = TemperatureConverter.FahrenheitToCelsius(70);
    System.assertEquals(21.11,celsius);
  }
  @isTest static void testFreezingPoint() {
    Decimal celsius = TemperatureConverter.FahrenheitToCelsius(32);
    System.assertEquals(0,celsius);
  }
  @isTest static void testBoilingPoint() {
    Decimal celsius = TemperatureConverter.FahrenheitToCelsius(212);
    System.assertEquals(100,celsius,'Boiling point temperature is not expected.');
  }
  @isTest static void testNegativeTemp() {
    Decimal celsius = TemperatureConverter.FahrenheitToCelsius(-10);
    System.assertEquals(-23.33,celsius);
  }
}
´´´
```

@RestResource(urlMapping='/Accounts/*/contacts')
global with sharing class AccountManager {
  @HttpGet
    global static Account getAccount() {
        RestRequest request = RestContext.request;

        // grab the caseId from the end of the URL
        String accId = request.requestURI.substringBetween('/Accounts/', '/contacts');
        Account result =  [SELECT Name,Id, 
						(SELECT LastName, Id FROM Contacts)
                        FROM Account
                        WHERE Id = :accId];
        return result;
    }
}
´´´
```

@IsTest
private class AccountManagerTest {
    @isTest static void testGetAccById() {
   Id recordId = getTestRecord();
        // Set up a test request
        RestRequest request = new RestRequest();
        request.requestUri =
            'https://yourInstance.my.salesforce.com/services/apexrest/Accounts/' + recordId + '/contacts';
        request.httpMethod = 'GET';
        RestContext.request = request;
        
        // Call the method to test
        Account thisAcc = AccountManager.getAccount();
        
        // Verify results
        System.assert(thisAcc != null);
        System.assertEquals('Test record', thisAcc.Name);
    }
    private static Id getTestRecord(){
        Account acc = new Account(Name='TestAccount');
        Insert acc;
        Contact con = new Contact(LastName='Annson', AccountId = acc.Id);
        Insert con;
        return acc.Id;
    }

}
```
