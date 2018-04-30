---
title: Filtering Magazines Issues with AJAX
layout: post
---

While working at Vance Publishing, I ran into a few challenges that made work more interesting to me.
There was a good patch of maybe 2-3 months where it felt like I was doing the same thing *everyday* (which I was), and it was getting old.
So I appreciated the chance to take on the following task:

> ## THREAT: Magazine Issue Filtering!
> The fashion-forward fanatics can't find fastly. (The salon magazine 
> clients wanted to be able to get to their magazine issues more quickly)

> ## TASK: Simplify the process 
> * Create a filtering interface to fetch magazines from one location at the top of the archive.
> * The interface should be a series of 3 drop-down selects, each filtering further down to a final, single magazine selection.
> * After the final selection is made, a "Go" button would navigate directly to that issue's page.

To get the data properly, we set up the first drop down to dynamically represent all of the 
magazines in the archive that would have even one issue available. The options in this drop down were assigned values equal to 
that magazine's id in the service we set up using [Services Views](https://www.drupal.org/project/services_views) for Drupal.
It allowed us to turn a Drupal "View" into an API.

We used this to make a call to that service and return HTML data to populate the next two drop downs, the third (the issue)
depending on the selection in the second dropdown. After all three (and *only* after all three) are selected, 
then pressing 'Go' will navigate directly to the magazine article you chose. 


```
  function getDropDownData (value){
    $('option.delete-me').remove();
    selectYear.prop('disabled', 'disabled');
    selectIssue.prop('disabled', 'disabled');
    ajaxResult = $.ajax(
        '/api/views/magazine_collection?tid_raw=' + value,
        {
          dataType: "json",
          dataFilter: function(data, type){
            parsed_data = $.parseJSON(data);
            $.each(parsed_data, function(i, item){
              date = item.issue_date;
              temp_data = date.slice(0, date.indexOf(' ')).split('-');
              item.date_data = {
                year: temp_data[0],
                month: temp_data[1],
                day: temp_data[2]
              };
            });
            return JSON.stringify(parsed_data);
          },
          type: "GET"
        });

    ajaxResult.done(function(data){
      years = [];

      $.each(data, function(i, item){
        if(years.indexOf(item.date_data['year']) === -1){
          years.push(item.date_data['year']);
        }
      });
      years.sort().reverse();
      selectYear.removeProp('disabled');

      $.each(years, function(y, year){
          selectYear.append(
              [year option assembled with data and jQuery]
          );
      });

      selectYear.change(function(){
        findIssuesByYear(this.value, data);
      });
    });

    ajaxResult.fail(function(){
      console.log('You embarrass me!');
    });
  }

  function findIssuesByYear(value, data){
    issues = [];
    $.each(data, function(i, item){
      if(item.date_data['year'] == value){
        issues[item.date_data.month - 0] = item;
      }
    });

    selectIssue.removeProp('disabled');
    selectIssue.find('.delete-me').remove();

    $.each(issues, function(i, issue){
      if(issue){
        selectIssue.append(
            [issue option assembled with data and jQuery]
        );
      }
    });
  }

  function setNavigation(){
    $('#go-to-issue').attr('action', selectIssue.val());
  }
```