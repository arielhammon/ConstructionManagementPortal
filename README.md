# Construction Management Portal

## Introduction
This file contains descriptions and code samples of work I did on a private company portal for the construction firm [Erectors Inc.](http://erectorsinc.com) based in Portland, Oregon. The portal featured company news, chat, scheduling of jobs and employees, and management of user roles (permissions). It was an ASP.NET MVC, C#, code-first database application.

Since I joined the development team late in the cycle, my work primarily fell into three areas:
1. remodeling and refactoring, 
2. bug fixes, and 
3. adding robustness to existing routines. 

As a developer who loves building elegant code, it was a challenge to enter the project so close to its delivery date. I learned to set aside some of my perfectionist tendancies in order to help the team meet the delivery date. While working on my stories, I encountered many previous samples of work that I thought needed to be reworked. Working closely with the project manager, I sought approval to improve areas that I could effectively manage on a daily cycle. Working in this mode, I was able to complete nearly one story per day. Some were more difficult than others; some spannned many different areas (files) of code while others were more localized. I will highlight a few below along with some other [team based skills](#team-based-skills) which I demonstrated during my work on this project.

## Contents
Sample Stories Completed
1. [Remodel Views and Controllers Showing Employee Schedules](#remodel-views-and-controllers-showing-employee-schedules)
2. [Remodel ShiftTime Class](#remodel-shifttime-class)
3. [Fix Employee Filter Criteria](#fix-employee-filter-criteria)
4. [Change Employee Status Confimation](#change-employee-status-confimation)
5. [Two Part Delete Confirmation](#two-part-delete-confirmation)

[Some Other Stories Completed](#some-other-stories-completed)

[Team Based Skills](#team-based-skills)

## Sample Stories Completed

### Remodel Views and Controllers Showing Employee Schedules
#### Original State (brief description)
Originally, the SchedulesController and corresponding Views were modeled on a generic list containing ALL employee schedules retrieved from the database. In order to show a particular employee his/her schedule, the View would sort through ALL schedules to find only current items for that particular employee, create an instance of the JobsController, and then retrieve the job details from the JobsController. Not only was this ineficient, but it violated one of the foundational principles of MVC, separation of concerns.
#### Remodeled As Follows
A previous developer had already begun remodeling the Controller/Views based on a Dictionary of <Job, Schedules>. However, it still retrieved all schedules from the database. So the first thing I did was add a method to filter for specific schedules based on optional parameters. As I mentioned earlier, I have a passion for writing elegant and efficient code, but sometimes this affects readability. Therefore, I thoroughly commented my work and included a table for quick reference to describe how the method handled null (or missing) values.

	public List<Schedule> GetSchedules(DateTime? startDate = null, DateTime? endDate = null, 
										string UserId = null, Guid? JobId = null)
	{
		//Suppose that [--------] represents the span (start and end dates) of a job:
		//We want to include scenarios 1-4 in the query and not scenarios 5-6, but
		//the negation of (scenarios 1-4 AND not scenarios 5-6) == (not scenarios 1-4 OR scenarios 5-6).
		//Calculating scenarios 5-6 is shorter and much faster for large queries.
		//Hence, we will include the negation of scenarios 5-6 which will give us scenarios 1-4.
		//I will include all coded scenarios for refernce.  --Ariel Hammon 7/17/2019
		//
		//              startDate       endDate
		//                  |               |
		//              [---|---------------|---]               scenario 1
		//              [---|------------]  |                   scenario 2
		//                  |   [-----------|---]               scenario 3
		//                  |   [-------]   |                   scenario 4
		//                  |               |
		//  [----------]    |               |                   scenario 5
		//                  |               |   [----------]    scenario 6
		//
		//If any of the optional parameters are missing or null, that parameter is ignored in the query. To be specific:
		//      startDate   endDate
		//      =========   =======
		//      null        null    :   include all x regardless of x.StartDate or x.EndDate
		//      null        !null   :   incude all x where x.EndDate <= endDate
		//      !null       null    :   include all x where x.StartDate >= startDate
		//

		List<Schedule> schedules = db.Schedules.Where(x => (UserId == null || x.Person.Id == UserId) &&
				(JobId == null || x.Job.JobId == JobId) &&
				!((startDate != null && x.EndDate != null && x.EndDate < startDate) || (endDate != null && x.StartDate > endDate)))
				.OrderBy(x => x.StartDate).ThenBy(x => x.EndDate).ThenBy(x => x.Person.UserName).ToList();

		//The above is equivalent to this, but as you can see, the above requires a lot fewer comparisons:
		//List<Schedule> schedules = db.Schedules.Where(x => (UserId == null || x.Person.Id == UserId) &&
		//  (JobId == null || x.Job.JobId == JobId) &&
		//  (((startDate == null || x.StartDate <= startDate) && (endDate == null || x.EndDate >= endDate))     ||  //scenario 1
		//   ((startDate == null || x.StartDate <= startDate) && (endDate == null || x.EndDate >= startDate))   ||  //scenario 2
		//   ((startDate == null || x.StartDate <= endDate) && (endDate == null || x.EndDate >= endDate))       ||  //scenario 3
		//   ((startDate == null || x.StartDate >= startDate) && (endDate == null || x.EndDate <= endDate))))       //scenario 4
		//  .OrderBy(x => x.StartDate).ThenBy(x => x.EndDate).ThenBy(x => x.Person.UserName).ToList();

		return schedules;
	}

While at it, I noticed that the SchedulesController, which created an instance of the database context, did not have a dispose method, so I added that. I only mention this minor (but very important) modification to demonstrate that I am attentive to details.

	protected override void Dispose(bool disposing)
	{
		if (disposing)
		{
			db.Dispose();
		}
		base.Dispose(disposing);
	}

The next issue to address was that the Views needed some information about Jobs, such as non-default start times, that was not contained in the Job object. This was one reason that the Views incorrectly created an instance of the JobsController. I recommended that we create a ViewModel to contain this information and pass it to the View. However, the project manager wanted an AJAX call instead, so that's what I did.

	//AJAX client side call from the View
	function DoNonDefaultStartTimesAlert() {
		var jobIDs = []; //array of strings containing JobIds
		$("[id^='JobID:']").each(function () {
			jobID = this.id;
			jobID = jobID.replace("JobID:", "");
			jobIDs.push(jobID);
		});
		var xhttp = new XMLHttpRequest();
		xhttp.onreadystatechange = function () {
			if (this.readyState == 4 && this.status == 200) {
				var pairsArray = JSON.parse(xhttp.response);
				for (i = 0; i < pairsArray.length; i++) {
					var pair = pairsArray[i];
					var element = document.getElementById('JobID:' + pair.ID);
					element.dataset.content = pair.Info;
					element.hidden = false;
					$(element).popover();
				}
			}
		}
		try {
			if (jobIDs.length > 0) {
				xhttp.open("GET", '/Jobs/FetchJobsNonDefaultStartTimes?JobIDs=' + jobIDs, true);
				xhttp.send();
			}
		}
		catch (ex) {
			document.getElementById('AJAXNonDefaultTimeWarning').innerHTML =
				"<p>There was an error retrieving non-default shift start times.</p>" +
				"<p>Some jobs may have non-default start times.</p>";
		}
	}

	//AJAX server side response
	[HttpGet]
	//currently used by the _MasterSchedulePartial view to get alert messages for jobs with non-default start times
	public JsonResult FetchJobsNonDefaultStartTimes(string jobIDs)
	{
		string[] jobIDsArray = jobIDs.Split(',');
		var result = new List<object>();
		foreach (string jobID in jobIDsArray)
		{
			Job job = db.Jobs.Find(Guid.Parse(jobID));
			if (job != null)
			{
				string message = GetNonDefaultStartTimeMessage(job);
				if (message != "")
				{
					var pair = new { ID = job.JobId.ToString(), Info = message };
					result.Add(pair);
				}
			}
		}
		return Json(result, JsonRequestBehavior.AllowGet);
	}

Finally, it remained to refactor the Views to utilize the new model. These were very lengthy, so I'll remove most of the details:

	@model Dictionary<Job, List<Schedule>>
	<div class="container" id="scheduleList">
		@foreach (KeyValuePair<Job, List<Schedule>> pair in Model)
		{
			Job job = pair.Key;
			...
				<tr>
				...
					<td style="width:30%">
						<span style="font-weight:bold">@job.JobTitle</span>
						@*StreetAddress and City are not assigned for PTO, Injured, etc.*@
						@if (!string.IsNullOrEmpty(job.StreetAddress) || !string.IsNullOrEmpty(job.City))
						{
							<br /><span>@job.StreetAddress @job.City, @job.State.GetDescription()</span>
						}
					</td>
					<td style="width:30%">
						@*Display an alert symbol to let the user know there is something different about this
							schedule. When hovered over, it will give the user extra information about what kind
							of changes there are.
							The following alert is initially hidden and will be unhidden if necessary and
							populated with content in the jquery function below in the script section.*@
						<a id="JobID:@job.JobId" href="#" title="Schedule Changes" data-html="true" data-toggle="popover"
							data-trigger="hover" data-placement="top" data-content="" hidden>
							<span class="glyphicon glyphicon-alert"></span>
						</a>
						...
					</td>
				...
				</tr>
				...
				@foreach (Schedule schedule in pair.Value)
				{
				...
					<tr>
					...
						<td style="width:20%">
							@*Only Display Phone # for Foreman*@
							@if (schedule.Person.WorkType == WorkType.Foreman)
							{
								<i class="glyphicon glyphicon-phone-alt"> </i> <a href="tel: +1 @schedule.Person.PhoneNumber">@schedule.Person.PhoneNumber</a>
							}
						</td>
						<td style="width:20%">
							@if (schedule.EndDate.HasValue)
							{
								@Convert.ToDateTime(schedule.EndDate).ToString("MM-dd-yyyy")
							}
						</td>
					...
					</tr>                        
				}
			</div>
		}
		<span id="AJAXNonDefaultTimeWarning"></span>
	</div>

*jump to: [Contents](#contents)*

### Remodel ShiftTime Class
#### Original State (brief description)
Originally, there were several Views that defined a helper method to convert 24 hour time to AM/PM time. This violated the DRY principle (do not repeat yourself) and created several inefficiencies. There were also two separate helper methods in the JobsController which duplicated calls to the database and created unnecessary if-statements.
#### Remodeled As Follows
I refactored these methods into a partial class and added another override method because there were several Views which were incorrectly converting ShiftTime objects to strings.

	public partial class ShiftTime
	{
		//Created partial class to isolate utility methods from the basic properties tied to the database.

		//Added these methods because there are a number of views (duplicated) which
		//create this method in loops (inefficient), and a number of views which
		//do not use any such method to display times (inconsistent). --Ariel Hammon 7/20/2019
		public static string ConvertToAMorPM(string time)
		{
			if (string.IsNullOrEmpty(time)) return time;
			try
			{
				string[] SplitTime = time.Split(':');
				if (SplitTime.Length != 2) return time;
				int hours = Convert.ToInt16(SplitTime[0]);
				string minutes = SplitTime[1];
				string meridian;
				if (hours > 12)
				{
					meridian = "PM";
					hours -= 12;
				}
				else if (hours == 12)
				{
					meridian = "PM";
				}
				else if (hours == 0)
				{
					hours = 12;
					meridian = "AM";
				}
				else
				{
					meridian = "AM";
				}
				time = Convert.ToString(hours) + ":" + minutes + " " + meridian;
				return time;
			}
			catch (Exception)
			{
				return time;
			}
		}

		public string[] NonDefaultShiftTimes()
		{
			//Refactored two old methods from the JobsController:
			//      string daysWithoutDefaultStartTime(Job job)
			//      public bool HasNonDefaultShiftTime(string jobNumber)
			//into this method and GetNonDefaultStartTimeMessage in the JobsController
			//in order to avoid extra if-statements and to avoid
			//unnecessary calls to the the database. --Ariel Hammon 7/19/2019

			var result = new List<string>(capacity: 2); //initial capacity

			if (Monday      != null && Monday       != Default) { result.Add("Monday: "     + ConvertToAMorPM(Monday));      }
			if (Tuesday     != null && Tuesday      != Default) { result.Add("Tuesday: "    + ConvertToAMorPM(Tuesday));     }
			if (Wednesday   != null && Wednesday    != Default) { result.Add("Wednesday: "  + ConvertToAMorPM(Wednesday));   }
			if (Thursday    != null && Thursday     != Default) { result.Add("Thursday: "   + ConvertToAMorPM(Thursday));    }
			if (Friday      != null && Friday       != Default) { result.Add("Friday: "     + ConvertToAMorPM(Friday));      }
			if (Saturday    != null && Saturday     != Default) { result.Add("Saturday: "   + ConvertToAMorPM(Saturday));    }
			if (Sunday      != null && Sunday       != Default) { result.Add("Sunday: "     + ConvertToAMorPM(Sunday));      }
			return result.ToArray();
		}

		//Added this overide because there were calls to convert instances of this
		//class to string representations which were not producing actual days and times.
		public override string ToString()
		{
			string result = "";
			if (!string.IsNullOrEmpty(Default)) result = "Default: " + ConvertToAMorPM(Default);
			string[] otherTimes = NonDefaultShiftTimes();
			for (int i = 0; i < otherTimes.Length; i++)
			{
				result += ", " + otherTimes[i];
			}
			return result;
		}
	}

*jump to: [Contents](#contents)*

### Fix Employee Filter Criteria
#### Original State (brief description)
Somewhere in the development process, a collection of filter boxes for the master list of employees (based on name, worktype, role, status, etc.) had become horribly broken.
#### Remodeled As Follows

	function filter() {
		var tableRows = document.getElementById("userList").getElementsByTagName("tr");
		//the headings row is the first (index==0) row in the table
		//the filter row is the second (index==1) row in the table
		var filterRowCells = tableRows[1].getElementsByTagName("td");
		//Get filter values in an array, convert to lowercase for comparison purposes
		var filterValues = [];
		for (var i = 0; i < filterRowCells.length; i++) {
			var element = filterRowCells[i].firstElementChild;
			if (element != null) { //there are some empty cells with no filterbox (ElementChild)
				filterValues.push(element.value.toLowerCase());
			} else {
				filterValues.push("");
			}
		}
		//Iterate over rows
		//the first data row to filter is row 3 (index==2)
		for (var i = 2; i < tableRows.length; i++) {
			var hits = [];  //A list of booleans, if any are false, hide the row.
			var dataRowCells = tableRows[i].getElementsByTagName("td");
			for (var j = 0; j < dataRowCells.length; j++) {
				var dataValue;
				var selectElements = dataRowCells[j].getElementsByTagName("select");
				if (selectElements.length > 0) {  //Filtering for drop-down list contents
					//The inner text of a <select> element is each option as plain text seperated by new line characters,
					//which we split into an array and take the first element which is the default/selected option.
					//Before a selection is explicityly made, .value is empty.
					dataValue = selectElements[0].innerText.split("\n")[0];
				}
				else {
					dataValue = dataRowCells[j].innerText;
				}
				var hit = dataValue.toLowerCase().includes(filterValues[j]);
				hits.push(hit);
			}
			if (hits.includes(false)) {
				tableRows[i].hidden = true;
			}
			else {
				tableRows[i].hidden = false;
			}
		}
	}

*jump to: [Contents](#contents)*

### Change Employee Status Confimation
#### Original State (brief description)
There were two controls (a dropdown box and a checkbox) that allowed Admins to change an employees worktype or status. There was also supposed to be a confirmation, but neither aspects were working properly. Also, they didn't take into account whether or not the status had actually changed. In other words, an Admin might get a message to confirm a change in status that hadn't actually changed.
#### Remodeled As Follows
The following sample of HTML code tracked the original values:

	<input type="submit" value="Submit" onclick="ConfirmChange('@item.UserRole.ToString()','Are you sure you want to change this user\'s role?','@editroleFormId','@ValueElemID')" />

Then the following JavaScript handled the confirmation:

	function ConfirmChange(originalState, msg, FormId, ValueElemId) {
		event.preventDefault();
		var valueElement = document.getElementById(ValueElemId);
		var currentState = "";
		switch (valueElement.tagName) {
			//use switch to make it easy to add other options in the future
			case "SELECT":
				currentState = valueElement.value;
				if (currentState == "") { //no selection has been explicitly made, so get the default value
					currentState = valueElement.innerText.split("\n")[0];
				}
				break;
			case "INPUT":
				switch (valueElement.type) {
					case "checkbox":
						currentState = valueElement.checked;
						break;
				}
		}
		currentState = currentState.toString();
		if (currentState == "") {
			modalInfo("Sorry, we couldn't read the state of this change.");
		} else if (originalState.toLowerCase() == currentState.toLowerCase()) {
			modalInfo("The user's status didn't change. If you want to change the user's status, alter the setting and resubmit.");
		} else {
			modConfirm(msg, FormId);
		}
		return false;
	}

*jump to: [Contents](#contents)*

### Two Part Delete Confirmation
#### Original State (brief description)
When a job was deleted, there was a confirmation, but the project manager wanted a two-part confirmation if the job had any current or future schedules associated with it.
#### Remodeled As Follows
I added the following JQuery script which took two different confirmation messages as input and the formID which submitted the command to the server. The most challenging aspect of this change was dealing with using a single HTML modal and managing the hide/show events which triggered almost simultaneously and caused the browser to hang. I worked around this problem by creating an event listener that triggered the second confirmation after the first confirmation had fully completed, on "hidden.bs.modal". I thought this was preferable to using two different HTML modals.

	//If the user clicks OK at first then the modal is shown again
	//and will submit the form on the second click of OK
	var modConfirmTwoPart = function (msg1, msg2, formId) {
		event.preventDefault();
		$("#btnOK").unbind('click');
		$("#btnCancel").unbind('click');
		$("#confirmModal").unbind("hidden.bs.modal");
		$("#confirmModal .modal-body").removeClass('text-danger');
		$("#confirmModal").unbind('shown.bs.modal');
		$("#btnOK").on("click", function () {
			$("#confirmModal").on("hidden.bs.modal", function () {
				DoPartTwo();
			});
			$("#confirmModal").modal('hide');
		});
		function DoPartTwo() {
			$("#btnOK").unbind('click');
			$("#confirmModal").unbind("hidden.bs.modal");
			$("#btnOK").on("click", function () {
				$("#confirmModal").modal('hide');
				$("#" + formId).trigger('submit');
			});
			$("#confirmModal .modal-body").addClass('text-danger').text(msg2);
			$("#confirmModal").modal('show');
		}
		$("#btnCancel").on("click", function () {
			$("#confirmModal").modal('hide');
		});
		$("#confirmModal .modal-body").text(msg1);
		$("#confirmModal").modal('show');
		return false;
	}

*jump to: [Contents](#contents)*

### Some Other Stories Completed

#### Assign Foreman on Job Creation
During the development cycle, the option to assign a foreman at the time that a job was being created got broken. I fixed this and also added some form validation JavaScript to alert the user of potential problems before sending the form to the server.

#### Replacing browser alerts with custom message modal
Previously, some developers had used the browser's alert command to convey information to the user. This was jolting because it had a completely different style and feel than the custom message modals in use elsewhere. I replaced all of these with the custom message modals and made the necessary HTML and Javascript changes.

*jump to: [Contents](#contents)*

## Team Based Skills
* Productivity Meetings. Participated in daily standup meetings to keep everyone updated on what we were working on, what we planned to work on next, and what obstacles we were encountering. This helped to focus our energies. For example, if I wanted to work on the same story as someone else, but the other person was closer to finishing their current assignment, I would work in a different direction. Knowing where everyone was working actually helped me increase my productivity by becoming the "expert" in a group of stories while other developers also improved their productivity by focusing on their group of stories.
* Mutual Assistance. Getting help from team-mates who could offer assistance with my obstacles or offering assistance to others when I could help them.
* Communication and Collaboration. When a couple of my stories involved files which overlapped with another developer's stories, we engaged in extensive communication in order to avoid conflicts. Not only did this reduce frustration, it also improved productivity because I knew where to focus my efforts while he finished up something that he was working on.

*jump to: [Contents](#contents)*
