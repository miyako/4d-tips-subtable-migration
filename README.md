# 4d-tips-subtable-migration
Moved from sources.4d.com

== Removing Subtables ==

This article explores how one can procedurally depart from the use of subtables in 4D v12. 

'''Prerequisite'''

There are a few critical conditions the original database must meet.

1.	That there are only single level subtables. If the structure contains multi level subtables, please refer to Technical Note [http://kb.4d.com/search/assetid=47983 Upgrading Subtables to 4D v11 SQL] on how to deal with such legacy assets.

2.	The the converted subtable name is unchanged. When a subtable field named [PARENT]CHILD is converted to v11 or above, it we automatically be transformed to a regular table with the name [PARENT_CHILD]. Our strategy depends on this specific naming convention, so it should remain unchanged for the operation to succeed.

3.	That the converted subtable key field name is unchanged. When a subtable field is converted as described above, there will be added a new field named id_added_by_converter. This name should also be kept intact.

'''Note''': The purpose of this tutorial is to demonstrate how one can take advantage of the various migration tools available in 4D v12, as well as better understand the specific limitations of the current compatibility mode. It is advised that one first test the method described using the sample database, before adapting the technique to real world situations. In any case, one should take reasonable precautions to counter unexpected consequences.

'''Downloads'''

[http://sources.4d.com/trac/4d_keisuke/browser/COMPONENT/SUBTABLE.zip Sample Database]

[http://sources.4d.com/trac/4d_keisuke/browser/COMPONENT/Subtable%20Tools.4dbase.zip Subtable Tools Component]

'''About the Sample Database'''

This v12 structure contains 1 table, which has a converted subtable field (originally created with 4D 2004). It has one dialog, on which you can navigate, query, sort, create, and delete subrecords. The "special relation" between the two tables, added by the converter, is unchanged. The goal is to cut this link permanently, without losing any of the existing features in the application.

[[Image(dialog.png)]]

Many of the deprecated commands are used in the dialog;

* ALL SUBRECORDS
* FIRST SUBRECORD
* LAST SUBRECORD
* PREVIOUS SUBRECORD
* NEXT SUBRECORD
* CREATE SUBRECORD
* DELETE SUBRECORD
* APPLY TO SUBSELECTION
* ORDER SUBRECORDS BY
* QUERY SUBRECORDS
* Records in subselection
* End subselection
* Before subselection

'''Note''': These commands are not used in this example;

* MODIFY SUBRECORD
* ADD SUBRECORD
* SEND RECORD
* RECEIVE RECORD

In addition, regular commands are used on the parent table, assuming that one of it's field is a subtable.

QUERY([Table1];[Table1]Field3'Field1=Get edited text+"@")[[BR]]
DUPLICATE RECORD([Table1]) `this ceased to work on subtables since migration[[BR]]

'''Step 1 - Reconfigure the Subform'''

If you have a subform widget placed on the parent form's input form, it's source would still be defined as the parent table's subtable field [PARENT]CHILD, even though the subtable has been renamed as [PARENT_CHILD] by the conversion. Both formats are acceptable while the special relations exist, but only the new table name will be valid once the link has been terminated, because the subtable field will then change type to long integer and no longer a valid data source. 

'''Before'''

[[Image(before.png)]]

'''After'''

[[Image(after.png)]]

If you change the subform's data source from the subtable field to the converted table, you will notice that the correct output form is displayed on the parent form, but its content will no longer reflect the current subselection. 

When a subtable is converted to a regular table linked to the parent table by the special relation, it is important to understand that the current selection of the related table and the subselection of the subtable field are NOT synchronized. In other words, loading a record in the parent table will automatically load the record's subselection, which are accessible via the legacy subtable commands and the legacy subtable notation, but the current selection of the related table is unchanged. Likewise, performing a query on the related table will have no effect on the parent table to which it is linked as a subtable. 

This means there can some ideological conflicts when one tries to mix the two concepts (compatibility subtable and related regular table) in code. For example, if the subselection was loaded as part of the parent record with read write access, by definition all of the subrecords are locked by the process and modifiable. However, since it is possible to access individual subrecords regardless of the parent table's current records by dealing with them as regular table records, it is therefore possible that some of the subrecords are locked by a different process to the process who is locking the parent record. This could never have happened in a real subtable, where all subrecords were part of the parent record to which it was related and only accessible through the parent table.

One of the benefits of using a subtable, was that one could open a parent record, add, delete, modify multiple subrecords, then decide whether to validate or not that entire operation. It was like a mini transaction. Using the legacy subtable commands in compatibility mode will still offer that convenience (with the exception of DUPLICATE RECORD), but using regular table commands on the converted table means you now have to explicitly start a transaction if changes to the ex-subtable should be treated as a whole.

'''The Subtable Tools Component'''

Now install the Subtable Tools component. This components provides a set of "wrapper" utility methods, designed with the following agenda;
 
* The method takes the same argument list as its legacy subtable couterpart.  
* The method performs the same operation, whether the special subtable link exists or not.
* The method internally uses only standard table commands.

The aim is to replace occurrences of the legacy subtable commands, prior to cutting the special link, in order to remove dependancies on deprecated commands as well as to prepare the application for the eventual removal of subtables altogether.

'''Using the wrapper methods'''

As mentioned earlier, changing the subform's data source from the subtable field to the converted table will inactivate the automatic loading of subrecords, as we are no longer requesting them using the subtable emulation layer. We need to explicitly manage the selection of the related table.

In our example with the subform, the subrecords were loaded automatically when we change the selection in the listbox that displays the parent table's selection,  

Here is the object method for the listbox.

{{{
If (Form event=On Selection Change) | (Form event=On Load)
 If (Records in set("ListboxSet0")=0)
  UNLOAD RECORD([Table1])
 Else 
  LOAD RECORD([Table1])
 End if
End if
}}}

LOAD RECORD or UNLOAD RECORD was the only action needed in this context; both commands would implicit perform an ALL SUBRECORDS, while would populate the subform widget with the current subselection. Now that the subform's data source is the related table and not the subtable field, we need to explicitly create a selection on that table.

Here is the modified object method.

{{{
If (Form event=On Selection Change) | (Form event=On Load)
 If (Records in set("ListboxSet0")=0)
  UNLOAD RECORD([Table1])
 Else 
  LOAD RECORD([Table1])
 End if
 TABLE_ALL_SUBRECORDS("[Table1]Field3")
End if
}}}

All of the component methods take table names and subtables names expressed as string. In the above example, the command will do the equivalent of ALL SUBRECORDS([Table1]Field3), regardless of whether the table has a subtable field attached via the special relation, or an ex-subtable that is now a regular table no longer linked to the parent table.

Now run the dialog again, and confirm that the subform is back in action thanks to the component method.

'''So how does it work?'''

There are several ways to access structure elements of the host database from within a component. One is to pass a pointer, which are always resolved by the callee (component) in the caller's (host's) context. Another is to refer to table and fields by their name, as done extensively in this component. It is possible to resolve a table or field in the host database by performing and SQL query on system tables in the component.  

The methods are private to the component, but you can check them out in the interpreted source code.

TABLE_Get_number

{{{
C_TEXT($1)
C_LONGINT($0)

ASSERT(Count parameters#0)

C_TEXT($tableName)
C_LONGINT($tableNumber)
ARRAY LONGINT($position;0)
ARRAY LONGINT($length;0)

$tableName:=$1

If (Match regex("\\p{Ps}?(\\P{Pe}+)";$tableName;1;$position;$length))

 $tableName:=Substring($tableName;$position{1};$length{1})

 $CS:=Get database parameter(SQL Engine Case Sensitivity)
 SET DATABASE PARAMETER(SQL Engine Case Sensitivity;0)

Begin SQL

 SELECT TABLE_ID FROM _USER_TABLES
 WHERE TABLE_NAME = :$tableName
 LIMIT 1
 INTO :$tableNumber

End SQL

SET DATABASE PARAMETER(SQL Engine Case Sensitivity;$CS)

$0:=$tableNumber

End if 

Method: TABLE_Get_pointer
}}}

TABLE_Get_pointer

{{{
C_TEXT($1)
C_POINTER($0)

ASSERT(Count parameters#0)

C_LONGINT($tableNumber)
$tableNumber:=TABLE_Get_number ($1)

If ($tableNumber#0)

 $0:=Table($tableNumber)

End if 
}}}

Notice the use of ASSERT. It is good practice to protect code segments by using this command extensively, especially in a component, where it is the responsibility of the component to verify that incoming data is valid. When an assertion fails, 4D will display a standard error dialog with the offending condition.

Here is the source code of the component method TABLE_ALL_SUBRECORDS which we inserted to the object method.

{{{
C_TEXT($1)

ASSERT(Count parameters#0)

C_POINTER($Subtable;$IdAddedByConverter;$SubrecordKeyField)
$Subtable:=TABLE_Get_subtable_pointer ($1)

ASSERT(Not(Nil($Subtable)))

$IdAddedByConverter:=TABLE_Get_id_added_by_converter ($1)

ASSERT(Not(Nil($IdAddedByConverter)))

$SubrecordKeyField:=FIELD_Get_pointer ($1)

ASSERT(Not(Nil($SubrecordKeyField)))

QUERY($Subtable->;$IdAddedByConverter->=Get subrecord key($SubrecordKeyField->))
}}}

The code does basically the following;

1. Resolve the string that is presumably a subtable field name. 
In our example "Table1_Field3" is resolved as ->[Table1]Field3. Incidentally, this is why you should not change the name of the converted subtable.

2. Get a pointer to the relation field added during conversion.
In our example we obtain ->[Table1_Field3]id_added_by_converter. Again, changing this field name will break our code.

3. Query the converted subtable by its key field (id_added_by_converter), with the value returned by Get subrecord key. This will effective perform an ALL SUBRECORDS on the related table.

Prior to v12, it was necessary to first remove the special link, which will make the legacy commands unusable, then create a new regular link, presumably with automatic relations, then add RELATE MANY to compensate for the loss of subtables. The advantage of v12 is that thanks to the new command Get subrecord key, it is possible to write code that works before and after the special link is removed, with or without a regular relation to replace it.

'''Note''': Get subrecord key is the only command that lets one read the private content of a field that is defined as a subtable relation link in a converted database. If the specified field is not a subtable relation link, it simply returns the long integer value of that field. Once a subtable relation link is removed, there is nothing to distinguish it from a standard long integer field.

'''Step 2 - Replace record navigation codes for the parent table'''

LOAD RECORD, the command used in the object method mentioned earlier, is not the only command that implicitly loads all subrecords for that parent record. Navigation commands such as FIRST RECORD, NEXT RECORD, etc. also have an effect on the subselectionsubselection. Since we are planning to abandon the use of legacy subtable commands, it is now necessary to manage the related selection explicitly.

One way of course is to add the component method TABLE_ALL_SUBRECORDS following every instance of a navigation command for the parent table, as well as any standard action with the same effect.

Alternatively, we can take advantage of the "Find in  Design" feature.

First, run the dialog and confirm that the "First Record" button is not working properly. Navigate to a record other than the first using the listbox, then click the button. The current record of the table does change, but not the related subselection displayed in the subtable.

Return to design mode, select "Find in Design" from the Edit menu.

[[Image(find_in_design.png)]]

'''Note''': Global search of project methods and variables can be started directly from the Method Editor's context menu. However, whether one can then proceed to global replacement depends on the context in which the dialog was called. Since we want to perform a replace of Language Expressions and/or Text (notably commands), we need to open the dialog from the edit Menu.

Select "Text" as the kind of object to find, type "FIRST RECORD([Table1])" as the criteria and click OK.

All matching occurrences are presented.

[[Image(found.png)]]

Click the cogwheel button at the bottom of the window and select "Replace in content".

[[Image(replace_in_content.png)]]

Enter "TABLE_FIRST_RECORD("[Table1]")" as the replacement text and click OK.

[[Image(replace.png)]]

All matching occurrences are replaced.

[[Image(replaced.png)]]

Run the dialog again and confirm that the "First Record" button now has effect on the ex-subtable.

The following wrapper methods are available in the component. You can likewise apply them to navigation commands that assume the loading of related subrecords. 

{{{
FIRST RECORD([Table1])
replace with: TABLE_FIRST_RECORD("[Table1]")

PREVIOUS RECORD([Table1])
replace with: TABLE_PREVIOUS_RECORD("[Table1]")

NEXT RECORD([Table1])
replace with: TABLE_NEXT_RECORD("[Table1]")

TABLE_LAST_RECORD([Table1])
replace with: TABLE_LAST_RECORD("[Table1]")
}}}

In addition, several common commands are also available as wrappers, that automatically handle the subselection for the current parent record. 

* TABLE_ALL_RECORDS
* TABLE_GOTO_SELECTED_RECORD
* TABLE_REDUCE_SELECTION

For all other cases (set, named selections, etc.) use these generic methods to load or unload the subselection related to the current parent record.

* TABLE_LOAD_RECORD
* TABLE_UNLOAD_RECORD

'''Note''': You may decide to not load subrecords automatically with the parent record, now that you have that option. In that case you leave existing code as there are and call TABLE_ALL_SUBRECORDS when necessary. However, as described earlier, this leave the code open to conflict where the subselection may be locked by another process that had forgotten to unload them with the parent record. It is the developers responsibility to avoid such scenario.

'''Step 3 - Replace record navigation codes for the related table'''

Similar to what we did with the parent table, replace occurrences of legacy subtable navigation commands with the methods provided by the component.

{{{
FIRST SUBRECORD([Table1]Field3)
replace with: TABLE_FIRST_SUBRECORD("[Table1]Field3")

PREVIOUS SUBRECORD([Table1]Field3)
replace with: TABLE_PREVIOUS_SUBRECORD("[Table1]Field3")

NEXT SUBRECORD([Table1]Field3)
replace with: TABLE_NEXT_SUBRECORD("[Table1]Field3")

LAST SUBRECORD([Table1]Field3)
replace with: TABLE_LAST_SUBRECORD("[Table1]Field3")
}}}

'''Step 4 - Replace other subtable commands for the related table'''

These relatively straightforward commands and functions can also be replaced globally.

{{{
ALL SUBRECORDS([Table1]Field3)
replace with: TABLE_ALL_SUBRECORDS("[Table1]Field3")

Records in subselection([Table1]Field3)
replace with: TABLE_Records_in_subselection("[Table1]Field3")

Before subselection([Table1]Field3)
replace with: TABLE_Before_subselection("[Table1]Field3")

End subselection([Table1]Field3)
replace with: TABLE_End_subselection("[Table1]Field3")
}}}

'''Note''': As par standard 4D behaviour, it is necessary for the ""Selection Mode" property of the subform of a regular table to be set to "Single" for the current record to be highlighted automatically. If the data source was a subtable, the current subrecord would be automatically highlighted even if the selection mode was set to "Multiple". 

'''Step 5 - Update code to query and sort subrecords'''

The next thing to consider is how to translate codes that work on subsections to regular table commands. As demonstrated in the sample database, there are 3 types of queries one would typically perform on a subselection.

* Query the parent table, which will implicitly load all the subrecord for the current record

QUERY([Table1];[Table1]Field2=value)

After conversion, the above line of code will only create a selection on the parent table, but not the related table, unless the special subtable relation link is kept or replaced by a regular automatic 1 to n relation.

* Query the parent table, based on the value of one of its subtable fields.

QUERY([Table1];[Table1]Field3'Field1=value)

After conversion, the above line of code will only work while the special subtable relation link is kept. Once the link is removed, the subfield notation [Table1]Field3'Field1 is no longer valid and must be replaced by a traditional query be formula (with related fields) or the v11.2 query by formula (using SQL JOIN).

* Query the subtable, to make a subset of the subrecords the new subselection.

QUERY SUBRECORDS([Table1]Field3;[Table1]Field3'Field1=value)

After conversion, the above line of code will only work while the special subtable relation link is kept. Once the link is removed, the subfield notation [Table1]Field3'Field1 is no longer valid. As described earlier, the current selection of the related table as a regular table is not tied to the parent table, meaning it must be managed using manual or automatic relations.

Again, the component offers wrapper methods to replace each of the 3 types of query.

{{{
QUERY([Table1];[Table1]Field2=value)
replace with: TABLE_QUERY("[Table1]";"[Table1]Field2=value")

QUERY([Table1];[Table1]Field3'Field1=value)
replace with: TABLE_QUERY("[Table1]";"[Table1]Field3'Field1=value")

QUERY SUBRECORDS([Table1]Field3;[Table1]Field3'Field1=value)
replace with: TABLE_QUERY_SUBRECORDS("[Table1]Field3";"[Table1]Field3'Field1=value")
}}}

'''So how that it work?'''

The wrapper command to QUERY must take into account 3 specific aspects related to subtables; that the search criteria may include the old subtable notation, and that the current selection of the related table must be updated if the criteria involves a field in that table (i.e. a cross table query). If the criteria does not directly involve the ex-subtable, then the operation is a regular query except for the fact that after the search the current selection for the related table needs to be updated.

Internally it calls a private method FORMULA_Translate, which will convert string such as ""[Table1]Field3'Field1" to "[Table1_Field3]Field1". Again, the prerequisite is that the converted table name has not been changed.

Then, it will parse the query criteria and extract the ex-subtable that in question, run a query using EXECUTE FORMULA, then relate the parent command by performing a second query based on the value of the id_added_by_converter and the subrecord key. 

As with all of the other component methods, the code is written to work before or after removing the special link, with or without a relation. 

The wrapper command to QUERY SUBRECORDS basically uses the same technique, but calls QUERY SELECTION to manage the subselection and does not relate back to the parent table.

'''Note''': As the exact purpose of a QUERY statement can be quite sensitive to the context in which it is called, it might be advisable to NOT resort to the global replace feature for these commands. Rather, you might want to use just the global find feature to search for occurrences then modify them on a case by case basis. Here are some examples:

{{{
QUERY([Table1];[Table1]Field3'Field1=Get edited text+"@")
replace with: 
$Get_edited_text:=Command name(655)
table_QUERY("[Table1]";"[Table1]Field3'Field1="+$Get_edited_text+"+\"@\"")

QUERY([Table1];[Table1]Field2=Get edited text+"@")
replace with: 
$Get_edited_text:=Command name(655)
table_QUERY("[Table1]";"[Table1]Field2="+$Get_edited_text+"+\"@\"")

$q:=Get edited text+"@"
QUERY SUBRECORDS([Table1]Field3;[Table1]Field3'Field1=$q)
replace with: 
$Get_edited_text:=Command name(655)
table_QUERY_SUBRECORDS("[Table1]Field3";"[Table1]Field3'Field1="+$Get_edited_text+"+\"@\"")
}}}

By the same token the component offers a method to replace the sort command.

{{{
ORDER SUBRECORDS BY([Table1]Field3;[Table1]Field3'Field1;<)
replace with: table_ORDER_SUBRECORDS_BY("[Table1]Field3";"[Table1]Field3'Field1;<")
}}}

'''Note''': The ORDER BY command accepts a variable number of arguments. In any case, you pass the entire parameter list as a single string.

'''Step 6 - Update commands to save and delete the parent record'''

'''Save'''

In compatibility mode, saving the parent record would automatically apply changes to the subtable, which is actually a related table. Now that we are departing from this roaming feature and working directly with the related table, we need to manage the subselection explicitly. More specifically, we need to save all modified records in the related selection of the related table individually. After that, we need to reproduce the current subselection (which might have been a subset of the full subselection, if QUERY SUBRECORDS was used) and reload the current record.

You can use the global find and replace feature with a corresponding component method. 

{{{
SAVE RECORD([Table1])
replace with: TABLE_SAVE_RECORD ("[Table1]")
}}}

'''Note''': Standard actions such as "Modify Subrecord", "Delete Subrecord" and "Create Subrecord" are optimised to interact with list type subforms as well as traditional list forms. If a subform has focus on a detail form, the action will apply to the data source associated to the subform. If the data source is a subtable, the modification, deletion and creation are temporary, that is, it is not validated until the parent record is saved. If the data source is a related table, the modification, deletion and creation are immediate. If the operation on the related table should not be validated until the parent record is saved, then one should consider starting a transaction.

'''Delete'''

In compatibility mode, saving the parent record would automatically delete all associated subrecords fetched via the special link. Now that we are departing from this roaming feature and working directly with the related table, we need to delete the subselection explicitly. 

You can use the global find and replace feature with a corresponding component method. 

{{{
DELETE RECORD([Table1])
replace with: TABLE_DELETE_RECORD ("[Table1]")
}}}

'''Step 7 - Update code to modify and delete subrecords'''

Now let's apply more wrappers for commands that modify and delete subrecords. In particular, we will remove the usage of these legacy commands.

{{{
APPLY TO SUBSELECTION([Table1]Field_3;[Table1]Field_3'VALUE:="def")
replace with: TABLE_APPLY_TO_SUBSELECTION ("[Table1]Field_3";"[Table1]Field_3'VALUE:=\"def\"")

DELETE SUBRECORD([Table1]Field_3)
replace with: TABLE_DELETE_SUBRECORD("[Table1]Field_3")

MODIFY SUBRECORD([Table1]Field_3;"Input";*)
replace with: TABLE_MODIFY_SUBRECORD ("[Table1]Field_3";"Input";"*")
}}}

Keep in mind that the new behaviour will not be strictly the same as the old code that work on subrecords (via the roaming link). Since we are now working on regular tables, the operation will instantly update the database. Consider starting a transaction if necessary.

The component also provides a utility method for wrapping assignments using the old subtable notation.

{{{
[Table1]Field_3'VALUE:="abc".
replace with: TABLE_EXECUTE_FORMULA ("[Table1]Field_3'VALUE:=\"abc\"")
}}}

'''Note''': Of course, you may choose to replace them in the method content and re-tokenize. 

Once we remove the special subtable relation, any code that uses the old notation will remain unchanged and prevent the application from being compiled. 

At this point, all of the commands listed below should have been replaced by their respective wrapper methods;

* ALL SUBRECORDS
* FIRST SUBRECORD
* LAST SUBRECORD
* PREVIOUS SUBRECORD
* NEXT SUBRECORD
* DELETE SUBRECORD
* APPLY TO SUBSELECTION
* ORDER SUBRECORDS BY
* QUERY SUBRECORDS
* Records in subselection
* End subselection
* Before subselection

'''Note''': In addition, you would have replaced any objects that have the "Delete Subrecord" standard action with TABLE_DELETE_SUBRECORD. 

However you might still have these legacy commands;

* ADD SUBRECORD
* CREATE SUBRECORD
* DUPLICATE RECORD (on the parent table)
* CREATE RECORD (on the parent table)

'''What does this all mean?'''

By using the component methods, we now deal directly on the current selection of the related table, instead of viewing them as part of the parent record. Because we changed the data source of the subform widget, it displays the current selection of the related table as if they were the parent table's subselection. Note that the subselection is managed by regular queries using the subrecords keys, which mean we are no longer dependent on the special subtable relation. In fact, we are ready to remove it in the structure editor.

You will have noticed that we have not yet touched on any commands that deals with the creation of new subrecords, such as ADD SUBRECORD or CREATE SUBRECORD. These have to be addressed after the special link is removed and here is the reason why. While the link is still in existence, the field type of the subrecord key and id_added_by_converter are read only. That is, you can't assign a new long integer value. The keys are managed by the compatibility layer. 

In order to create a new parent record, we need to assign a unique value to the subrecord key field. To create a new subrecord, we need to read the subrecord key of the parent record (which we can do with the new Get subrecord key command) and assign it to the id_added_by_converter field. Neither is possible unless we remove the special subtable link.

'''Step 8 - Time to remove the special relation!'''

The good news is that we are almost at the end of our migration. Open the structure editor, select the special relation that connects the two converted tables and delete.

'''Note''': This operation is not reversible.

Run the dialog and confirm that all of the subtable related features still work, even though we no longer have a link between the two tables. You don't have a create a new regular relation for the converted table to behave like a subtable.

Now go back to the Structure Editor and locate parent table's relation field, which has now turned into a regular long integer field. Activate the "Auto Increment" attribute on the property list.

The component provides two utility methods, one to check the largest registered subrecord key and another to check the largest sequence number for that table. In general, it would be easier to manage if the two were identical.

This method will update the table's sequence number so that it will be at least the same value as the largest subrecord key. By calling this method at least once after you have removed the special subtable relation, you ensure that the auto increment feature will yield a unique number that does not cause conflict with existing records.

{{{
TABLE_SYNC_SUBRECORD_KEY ("[Table1]Field_3")
}}}

At this point, you can finally replace codes that create new records, either on the parent table or the related ex-subtable.

{{{
CREATE RECORD([Table1])
replace with: TABLE_CREATE_RECORD ("[Table1]")

CREATE SUBRECORD([Table1]Field_3)
replace with: TABLE_CREATE_SUBRECORD ("[Table1]Field_3")
}}}

As a bonus, there is even a component method that duplicates the parent record with all its related subrecords.

{{{
DUPLICATE RECORD([Table1])
replace with: TABLE_DUPLICATE_RECORD ("[Table1]")
}}}

'''Note''': In compatibility mode, duplicating the parent record no longer created copies of the subrecords associated with that record. [http://kb.4d.com/search/assetid=47983 Upgrading Subtables to 4D v11 SQL] explains how one could duplicate subrecords using 4D v11 SQL. The following explanation takes advantage of the new v12 feature that is SQL EXPORT SELECTION.

'''Appendix i - SEND RECORD and RECEIVE RECORD'''

One popular usage of subrecords was to export and import the subselection with its parent record as one packet, using the commands SEND RECORD and RECEIVE RECORD. Now that the two tables are independent, this approach would be hard to replicate. In any case, v12 provides alternative methods to read and write records on disk. 

'''SQL EXPORT and SQL EXECUTE SCRIPT'''

The standard way to store records on disk to use these 2 powerful commands. They are extremely powerful, fast, standard based and compatible will all data types. However, it does not respect the logical integrity of the data, which is not quite the objective if you want to manage parent and child records as a whole.

'''USE EXTERNAL DATABASE'''

Another way to export and import records is to use an external database, again a new feature in v12. Unlike SEND RECORD and RECEIVE RECORD, you have random access and the full benefits of an SQL database. It does require complete refactoring, but for storing data to an external file in a logically consistent manner, this should be the preferred option.
