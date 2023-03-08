---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.10.3
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# (experimental) Badge Visualizations 


## Badge status 

```{warning}
this is not guaranteed to stay up to date with current, but the listed date is accurate. 
You can see the code that generates the figure behind the "click to show" button.  It is  Python code that you should be able to run anywhere with the required libraries installed. 
```

```{code-cell}
:tags: ["hide-input"]

%matplotlib inline
import os
from datetime import date,timedelta
import calendar
import pandas as pd
import numpy as np
import seaborn as sns
from myst_nb import glue
# style note: when I wrote this code, it was not all one cell. I merged the cells
# for display on the course website, since Python is not the main outcome of this course

# semester settings
first_day = date(2023,1,24)
last_day = date(2023,5,1)
meeting_days =[1,3] # datetime has 0=Monday
spring_break = (date(2023,3,11),date(2023,3,19))
penalty_free_end = date(2023, 2, 9)


# enumerate weeks
weeks = int((last_day-first_day).days/7+1)

# create differences
mtg_delta = timedelta(meeting_days[1]-meeting_days[0])
week_delta = timedelta(7)
skips = [date(2023,2,28)]
weeks = int((last_day-first_day).days/7+1)

meeting_days =[1,3]
mtg_delta = timedelta(meeting_days[1]-meeting_days[0])
week_delta = timedelta(7)

during_sb = lambda d: spring_break[0]<d<spring_break[1]


possible = [(first_day+week_delta*w, first_day+mtg_delta+week_delta*w) for w in range(weeks)]
weekly_meetings = [[c1,c2] for c1,c2 in possible if not(during_sb(c1))]
meetings = [m for w in weekly_meetings for m in w if not(m in skips)]


# build a table for the dates
badge_types = ['experience', 'review', 'practice']
target_cols = ['review_target','practice_target']
df_cols = badge_types + target_cols
badge_target_df = pd.DataFrame(index=meetings, data=[['future']*len(df_cols)]*len(meetings),
                                columns=df_cols).reset_index().rename(
                                   columns={'index': 'date'})
# set relative dates
today = date.today()
start_deadline = date.today() - timedelta(7)
complete_deadline = date.today() - timedelta(14)

# mark eligible experience badges
badge_target_df['experience'][badge_target_df['date'] <= today] = 'eligible'
# mark targets, cascading from most recent to oldest to not have to check < and >
badge_target_df['review_target'][badge_target_df['date'] <= today] = 'active'
badge_target_df['practice_target'][badge_target_df['date'] <= today] = 'active'
badge_target_df['review_target'][badge_target_df['date']
                          <= start_deadline] = 'started'
badge_target_df['practice_target'][badge_target_df['date']
                            <= start_deadline] = 'started'
badge_target_df['review_target'][badge_target_df['date']
                          <= complete_deadline] = 'completed'
badge_target_df['practice_target'][badge_target_df['date']
                            <= complete_deadline] = 'completed'
# mark enforced deadlines
badge_target_df['review'][badge_target_df['date'] <= today] = 'active'
badge_target_df['practice'][badge_target_df['date'] <= today] = 'active'
badge_target_df['review'][badge_target_df['date']
                          <= start_deadline] = 'started'
badge_target_df['practice'][badge_target_df['date']
                            <= start_deadline] = 'started'
badge_target_df['review'][badge_target_df['date']
                          <= complete_deadline] = 'completed'
badge_target_df['practice'][badge_target_df['date']
                            <= complete_deadline] = 'completed'
badge_target_df['review'][badge_target_df['date']
                          <= penalty_free_end] = 'penalty free'
badge_target_df['practice'][badge_target_df['date']
                          <= penalty_free_end] = 'penalty free'

# convert to numbers and set dates as index for heatmap compatibility
status_numbers_hm = {status:i+1 for i,status in enumerate(['future','eligible','active','penalty free','started','completed'])}
badge_target_df_hm = badge_target_df.replace(status_numbers_hm).set_index('date')

# set column names to shorter ones to fit better
badge_target_df_hm = badge_target_df_hm.rename(columns={'review':'review(e)','practice':'practice(e)',
                                                       'review_target':'review(t)','practice_target':'practice(t)',})
# build a custom color bar 
n_statuses = len(status_numbers_hm.keys())
manual_palette = [sns.color_palette("pastel", 10)[7],
                 sns.color_palette("colorblind", 10)[2],
                 sns.color_palette("muted", 10)[2],
                 sns.color_palette("colorblind", 10)[9],
                 sns.color_palette("colorblind", 10)[8],
                 sns.color_palette("colorblind", 10)[3]]
# generate the figure, with the colorbar and spacing
ax = sns.heatmap(badge_target_df_hm,cmap=manual_palette,linewidths=1)
# mote titles from bottom tot op
ax.xaxis.tick_top()
# pull the colorbar object for handling
colorbar = ax.collections[0].colorbar 
# fix the location fo the labels on the colorbar
r = colorbar.vmax - colorbar.vmin 
colorbar.set_ticks([colorbar.vmin + r / n_statuses * (0.5 + i) for i in range(n_statuses)])
colorbar.set_ticklabels(list(status_numbers_hm.keys()))
# add a title
today_string = today.isoformat()
glue('today',today_string,display=False)
glue('today_notdisplayed',"not today",display=False)
ax.set_title('Badge Status as of '+ today_string); 
```

The (e) columns are what will be enforced, the (t) columns is ideal, see the [deadlines section of the syllabus for more on the statuses](rp-status) for more detail. 

The following table shows, as of  {glue:}`today`, the number of badges with each status.  
```{code-cell}
:tags: ["hide-input"]

status_list_summary = ['eligible', 'penalty free', 'active', 'started', 'completed']
badge_count_df = badge_target_df[badge_target_df['date']<today].apply(pd.value_counts).loc[status_list_summary].drop(
                    columns='date').fillna(' ')

glue('exp_ontrack',int(badge_count_df.loc['eligible','experience']),display=False)
glue('exp_min',int(badge_count_df.loc['eligible','experience']-3),display=False)
glue('rp_completed',int(badge_count_df.loc['completed','review_target']),display=False)
glue('rp_started',int(badge_count_df.loc['started','review_target']),display=False)
badge_count_df
```

````{margin}
```{note}
This date is formatted like this because it comes from the code
```
```{important}
Remember, there will be 26 badge opportunities, but do not need all of them.  See the [](badgecounts) section of the syllabus for how they map into letter grades
```
````


This means that, as of {glue:}`today` you are fully caught up if you have:
-  {glue:}`exp_ontrack` experience badges submitted. 
-  {glue:}`rp_completed` total review and practice badges completed.
-  {glue:}`rp_started` additional review and/or practice badges started and in progress. 


Notes: 
-  If you do not have at least {glue:}`exp_min` experience badges earned, you should visit office hours to get caught up. 
-  There are are 6 review and practice badges that are from the penalty free zone, so they never expire. 
-  Most of your experience and completed badges should be earned, but there is no deadline to fix issues with them. 


## Prepare work and Experience Badges Process

This is for a single example with specific dates, but it is similar for all future dates

The columns (and purple boxes) correspond to branches in your KWL repo and the yellow boxes are the things that you have to do. The "critical" box is what you have to wait for us on. The arrows represent PRs (or a local merge for the first one)


```{mermaid}
sequenceDiagram
    participant P as prepare Feb 21
    participant E as experience Feb 23
    participant M as main 
    note over P: complete prepare work<br/> between feb 21 and 23
    note over E: run experience badge workflow <br/> at the end of class feb 23
    P ->> E: local merge or PR you that <br/> does not need approval
    note over E: fill in experience reflection 
    critical Badge review by instructor or TA
        E ->> M: Experience badge PR
    option if edits requested
        note over E: make requested edits
    option when approved 
        note over M: merge badge PR
    end
```

In the end the commit sequence for this will look like the following:

```{mermaid}
gitGraph
   commit
   commit
   checkout main
   branch prepare-2023-02-21
   checkout prepare-2023-02-21
   commit id: "gitunderstanding.md"
   branch experience-2023-02-23
   checkout experience-2023-02-23
   commit id: "initexp"
   merge prepare-2023-02-21
   commit id: "fillinexp"
   commit id: "revisions" tag:"approved"
   checkout main
   merge experience-2023-02-23
```

Where the "approved" tag represents and approving reivew on the PR. 


## Review and Practice Badge 


Legend:
```{mermaid}
flowchart TD
    badgestatus[[Badge Status]]
    passive[/ something that has to occur<br/> not done by student /]      
    student[Something for you to do]

    style badgestatus fill:#2cf

    decisionnode{Decision/if}
      sta[action a]
      stb[action b]
      decisionnode --> |condition a|sta
      decisionnode --> |condition b|stb
    subgraph phase[Phase]
      st[step in phase]
    end
```

This is the general process for review and practice badges 

```{mermaid}
flowchart TD
%%    subgraph work[Steps to complete]
    subgraph posting[Dr Brown will post the Badge]
      direction TB
      write[/Dr Brown finalizes tasks after class/]
      post[/Dr. Brown pushes to github/]
      link[/notes are posted  with badge steps/]
      posted[[Posted: on badge date]]
      write -->post
      post -->link
      post --o posted
    end
    subgraph planning[Plan the badge]
      direction TB
      create[/Dr Brown runs your workflow/]
      decide{Do you need this badge?}
      close[close the issue]
      branch[create branch]
      planned[[Planned: on badge date]]
      create -->decide
      decide -->|no| close
      decide -->|yes| branch
    create --o planned
    end
    subgraph work[Work on the badge]
      direction TB
      start[do one task]
      commit[commit work to the branch]
      moretasks[complete the other tasks]
      ccommit[commit them to the branch]
      reqreview[request a review]
      started[[Started <br/> due within one week <br/> of posted date]]
      completed[[Completed <br/>due  within two weeks <br/> of posted date]]
      wait[/wait for feedback/]
      start --> commit
      commit -->moretasks
      commit --o started
      moretasks -->ccommit
      ccommit -->reqreview
      reqreview --> wait
      reqreview --o completed
    end
    subgraph review[Revise your completed badges]
      direction TB
      prreview[Read review feedback]
      approvedq{what type of review}
      merge[Merge the PR]
      edit[complete requested edits]
      earned[[Earned <br/> due by final grading]]
      discuss[reply to comments]
      prreview -->approvedq
      approvedq -->|changes requested|edit
      edit -->|last date to edit: May 1| prreview
      approvedq -->|comment|discuss
      discuss -->prreview
      approvedq -->|approved|merge
      merge --o earned
    end

    posting ==> planning
    planning ==> work
    work ==> review

%% styling 
style earned fill:#2cf
style completed fill:#2cf
style started fill:#2cf
style posted fill:#2cf
style planned fill:#2cf
```


## Explore Badges


```{mermaid}
flowchart TD
    subgraph proposal[Propose the Topic and Product]
      issue[create an issue]
      proposed[[Proposed]]
      reqproposalreview[Assign it to Dr. Brown]
      waitp[/wait for feedback/]
      proceedcheck{Did Dr. Brown apply a proceed label?}
      branch[start a branch]
      progress[[In Progress ]]
      iterate[reply to comments and revise]
      issue --> reqproposalreview
      reqproposalreview --> waitp
      reqproposalreview --> proposed
      waitp --> proceedcheck
      proceedcheck -->|no| iterate
      proceedcheck -->|yes| branch
      branch --> progress
      iterate -->waitp
    end
    subgraph work[Work on the badge]
      direction TB
      moretasks[complete the work]
      ccommit[commit work to the branch]
      reqreview[request a review]
      wait[/wait for feedback/]
      complete[[Complete]]
      moretasks -->ccommit
      ccommit -->reqreview
      reqreview --o complete
      reqreview --> wait
    end
    subgraph review[Revise your work]
      direction TB
      prreview[Read review feedback]
      approvedq{what type of review}
      revision[[In revision]]
      merge[Merge the PR]
      edit[complete requested edits]
      earned[[Earned <br/> due by final grading]]
      prreview -->approvedq
      approvedq -->|changes requested|edit
      edit --> prreview
      edit --o revision
      approvedq -->|approved| merge
      merge --o earned
    end

    proposal ==> work
    work ==> review

%% styling 
style proposed fill:#2cf
style progress fill:#2cf
style complete fill:#2cf
style revision fill:#2cf
style earned fill:#2cf

```



## Build Badges 


```{mermaid}
flowchart TD
    subgraph proposal[Propose the Topic and Product]
      issue[create an issue]
      proposed[[Proposed]]
      reqproposalreview[Assign it ]
      waitp[/wait for feedback/]
      proceedcheck{Did Dr. Brown apply a proceed label?}
      branch[start a branch]
      progress[[In Progress ]]
      iterate[reply to comments and revise]
      issue --> reqproposalreview
      reqproposalreview --> waitp
      reqproposalreview --> proposed
      waitp --> proceedcheck
      proceedcheck -->|no| iterate
      proceedcheck -->|yes| branch
      branch --> progress
      iterate -->waitp
    end
    subgraph work[Work on the badge]
      direction TB
      commit[commit work to the branch]
      moretasks[complete the work]
      draftpr[Open a draft PR and <br/> request a review]
      ccommit[incorporate feedback]
      reqreview[request a review]
      wait[/wait for feedback/]
      complete[[Complete]]
      commit -->moretasks
      commit -->draftpr
      draftpr -->ccommit
      moretasks -->reqreview
      ccommit -->reqreview
      reqreview --> complete
      reqreview --> wait
    end
    subgraph review[Revise your work]
      direction TB
      prreview[Read review feedback]
      approvedq{what type of review}
      revision[[In revision]]
      merge[Merge the PR]
      edit[complete requested edits]
      earned[[Earned <br/> due by final grading]]
      prreview -->approvedq
      approvedq -->|changes requested|edit
      edit --> prreview
      edit -->revision
      approvedq -->|approved| merge
      merge --o earned
    end

    proposal ==> work
    work ==> review

%% styling 
style proposed fill:#2cf
style progress fill:#2cf
style complete fill:#2cf
style revision fill:#2cf
style earned fill:#2cf

```

      
## Tentative Alternative Calculation

```{warning}
Officially what is on the [](grading) page is what applies.  What follows is close to equivalent, but changes some thresholds. This is a *proposed* alternative, that I will adopt
if it is more clear and students agree it is fair. 
```

The total influence of a badge on your grade is the product of the badge's weight and its complexity.  All learning badges have a weight of 1, but have varying complexity.  All community badges have a complexity of 1, but the weight of a community badge can vary depending on what learning badges you earn. 

There are also some hard thresholds on learning badges. 
You must have:
- 24 experience badges to earn a D or above
- at least 18 additional total across reivew and practice to earn above C or above

Only community badges can make exceptions to these thresholds. So if you are missing learning badges required to get to a threshold, your community badges will fill in for those.  If you meet all of the thresholds, the community badges will be applied with more weight to give you a step up (eg C to C+ or B+ to A-). 

Community badges have the most weight if you are on track for a grade between D and B+  

If you are on track for an A, community badges can be used to fill in for learning badges, so for example, at the end of the semester, you might be able to skip some the low complexity learning badges (experience, review, practice) and focus on your high complexity ones to ensure you get an A. 

More precisely the order of application for community badges: 
- to make up missing experience badges 
- to make up for missing review or practice badge to reach a total of 18 between these two types
- to upgrade review to practice to meet a threshold
- to give a step up (highest weight)




<div>                        <script type="text/javascript">window.PlotlyConfig = {MathJaxConfig: \'local\'};</script>\n        <script src="https://cdn.plot.ly/plotly-2.12.1.min.js"></script>                <div id="badge_grade_graph" class="plotly-graph-div" style="height:700px; width:100%;"></div>            <script type="text/javascript">                                    window.PLOTLYENV=window.PLOTLYENV || {};                                    if (document.getElementById("badge_grade_graph")) {                    Plotly.newPlot(                        "badge_grade_graph",                        [{"alignmentgroup":"True","customdata":[["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",22,"D-"],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D "],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",24,"D+"],["experience",21,"C "],["experience",22,"C "],["experience",22,"C "],["experience",21,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",21,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",22,"C "],["experience",22,"C "],["experience",21,"C "],["experience",22,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",22,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",24,"C "],["experience",21,"C "],["experience",22,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",24,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",21,"C "],["experience",24,"C "],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"C+"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",24,"B-"],["experience",23,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",23,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",24,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",23,"B-"],["experience",24,"B-"],["experience",24,"B-"],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",23,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",24,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",24,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",24,"B "],["experience",20,"B "],["experience",24,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",20,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",23,"B "],["experience",23,"B "],["experience",23,"B "],["experience",23,"B "],["experience",23,"B "],["experience",24,"B "],["experience",24,"B "],["experience",23,"B "],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"B+"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A-"],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "],["experience",24,"A "]],"hovertemplate":"badge_det=experience<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"experience","marker":{"color":"#636efa","pattern":{"shape":""}},"name":"experience","offsetgroup":"experience","orientation":"v","showlegend":true,"textposition":"auto","x":["0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0cem","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","1cem","0cemrm","0cemrm","1cem","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","1cem","0cemrm","0cemrm","0cemrm","0cemrm","1cem","1cem","1cem","0cemrm","0cemrm","1cem","0cemrm","1cem","1cem","1cem","1cem","1cem","1cem","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1cem","0cemrm","1cem","1cem","1cem","1","1cem","1cem","1cem","1cem","1","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","2","1cp","2","2","1cp","1cp","1cp","2","1cp","1cp","1cp","1cp","2","2","2","2","2","2","2","1cp","2","2","2","2","2","2","2","2","2","2","1cp","1cp","2","2","1cp","2","1cp","3","4","3","3","3","3","3","3","3","4","3","3","3","4","3","3","3","3","3","3","3","3","3","3","3","4","4","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","4","2cem","4","4","4","4","4","4","3","4","4","4","4","4","4","4","2cem","4","4","4","2cem","2cem","2cem","2cem","2cem","4","2cem","2cem","2cem","2cem","2cem","4","3","5","5","1cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","6","6","6","6","6","5","0cemrupm","0cemrupm","1cemrupm","1cemrupm","6","1cemrupm","1cemrupm","0cemrupm","1cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","1cemrupm","1cemrupm","6","6","5","5","5","5","5","5","5","5","6","6","6","5","6","5","6","6","6","6","6","5","6","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","6","6","6","6","6","6","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","4cem","3cem","4cem","4cem","4cem","3cem","1cemrm","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","4cem","4cem","0cemrupm","3cem","3cem","3cem","1cemrm","3cem","1cemrm","3cem","3cem","3cem","3cem","5","5","5","5","5","5","5","1cemrupm","1cemrupm","1cemrupm","1cemrupm","5","5","1cemrupm","1cemrupm","0cemrupm","1cemrm","1cemrupm","1cemrupm","1cemrupm","5","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrm","1cemrm","1cemrupm","2cp","2cp","2cp","2cp","3cp","3cp","3cp","3cp","2cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","2cp","3cp","3cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","3cp","3cp","2cp","2cp","2cp","2cp","2cp","2cp","3cp","2cp","3cp","3cp","3cp","3cp","3cp","3cp","2cp","2cp","3cp","3cp","5cp","5cp","7","5cp","5cp","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","9","10","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","10","5cp","9","9","9","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","13","13","13","13","13","13","13","13","13","13","13","14","14","13","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","13","8","13","13","13","13","13","13","13","13","13","13","13","8","4cp","14","8","12","12","12","12","12","12","12","12","12","12","12","12","12","12","12","12","12","12","8","8","8","8","8","8","11","11","12","12","12","11","11","12","11","12","12","11","11","11","11","11","11","11","8","11","8","14","14","14","14","14","14","8","14","14","14","14","11","14","14","14","14","14","14","14","14","14","14","14","8","8","8","8","8","8","8","8","8","8","8","8","11","11","11","11","11","11","11","11","11","11"],"xaxis":"x","y":[2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0,2.0],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["community",6,"D-"],["community",6,"D-"],["community",6,"D-"],["community",6,"D-"],["community",6,"D-"],["community",6,"D-"],["community",9,"C "],["community",9,"C "],["community",9,"C "],["community",6,"C "],["community",9,"C "],["community",6,"C "],["community",6,"C "],["community",6,"C "],["community",9,"C "],["community",9,"C "],["community",6,"C "],["community",9,"C "],["community",9,"C "],["community",9,"C "],["community",6,"C "],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",11,"B-"],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",12,"B "],["community",3,"B "],["community",3,"B "],["community",3,"B "],["community",12,"B "]],"hovertemplate":"badge_det=community_experience_makeup<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"community_experience_makeup","marker":{"color":"#EF553B","pattern":{"shape":""}},"name":"community_experience_makeup","offsetgroup":"community_experience_makeup","orientation":"v","showlegend":true,"textposition":"auto","x":["0cem","0cem","0cem","0cem","0cem","0cem","1cem","1cem","1cem","0cemrm","1cem","0cemrm","0cemrm","0cemrm","1cem","1cem","0cemrm","1cem","1cem","1cem","0cemrm","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","1cemrupm","1cemrupm","1cemrupm","3cem"],"xaxis":"x","y":[0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666,0.6666666666666666],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",18,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",17,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",17,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",18,"C "],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",18,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",24,"C+"],["review",18,"C+"],["review",24,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"C+"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",18,"B-"],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",10,"B "],["review",10,"B "],["review",10,"B "],["review",10,"B "],["review",10,"B "],["review",10,"B "],["review",10,"B "],["review",10,"B "],["review",12,"B "],["review",12,"B "],["review",10,"B "],["review",10,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",12,"B "],["review",1,"B "],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"B+"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",12,"A-"],["review",6,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",6,"A "],["review",6,"A "],["review",12,"A "],["review",12,"A "],["review",12,"A "],["review",18,"A "],["review",12,"A "],["review",6,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",18,"A "],["review",6,"A "],["review",6,"A "]],"hovertemplate":"badge_det=review<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"review","marker":{"color":"#00cc96","pattern":{"shape":""}},"name":"review","offsetgroup":"review","orientation":"v","showlegend":true,"textposition":"auto","x":["1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","1cem","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","0cemrm","1cem","0cemrm","0cemrm","0cemrm","0cemrm","1cem","1","1","1","0cemrm","1","1","1","1","1","1","1","1","1","1","1","1","1","1","1","2","2","2","2","2","2","2","2","2","2","2","2","2","2","2","2","1cp","2","2","2","2","2","2","2","1cp","2","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","4","4","4","4","4","4","4","4","4","4","4","4","4","4","4","4","4","4","3","3","3","3","3","3","3","3","3","3","3","3","3","3","3","3","3","3","6","6","6","6","6","6","6","6","6","6","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","6","6","1cemrm","1cemrm","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","1cemrupm","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","12","13","13","13","13","13","13","13","13","12","12","13","13","13","11","13","12","11","11","11","11","11","11","11","11","11","11","11","11","11","11","11","11","11","12","12"],"xaxis":"x","y":[3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0,3.0],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["community",4,"C "],["community",4,"C "],["community",4,"C "],["community",4,"C "],["community",8,"B "],["community",8,"B "],["community",8,"B "],["community",8,"B "],["community",8,"B "],["community",8,"B "],["community",8,"B "],["community",8,"B "]],"hovertemplate":"badge_det=community_review_makeup<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"community_review_makeup","marker":{"color":"#ab63fa","pattern":{"shape":""}},"name":"community_review_makeup","offsetgroup":"community_review_makeup","orientation":"v","showlegend":true,"textposition":"auto","x":["0cemrm","0cemrm","0cemrm","0cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm"],"xaxis":"x","y":[0.75,0.75,0.75,0.75,0.75,0.75,0.75,0.75,0.75,0.75,0.75,0.75],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["community",3,"B "],["community",3,"B "],["community",3,"B "]],"hovertemplate":"badge_det=community_review_upgrade<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"community_review_upgrade","marker":{"color":"#FFA15A","pattern":{"shape":""}},"name":"community_review_upgrade","offsetgroup":"community_review_upgrade","orientation":"v","showlegend":true,"textposition":"auto","x":["1cemrupm","1cemrupm","1cemrupm"],"xaxis":"x","y":[1.0,1.0,1.0],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["practice",6,"B-"],["practice",6,"B-"],["practice",6,"B-"],["practice",6,"B-"],["practice",6,"B-"],["practice",6,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",16,"B-"],["practice",12,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",18,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",16,"B "],["practice",16,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",12,"B "],["practice",16,"B "],["practice",12,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",16,"B "],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",12,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",12,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",18,"B+"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",18,"A-"],["practice",24,"A-"],["practice",24,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",12,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",18,"A-"],["practice",12,"A "],["practice",12,"A "],["practice",12,"A "],["practice",12,"A "],["practice",12,"A "],["practice",6,"A "],["practice",12,"A "],["practice",12,"A "],["practice",12,"A "],["practice",12,"A "],["practice",6,"A "],["practice",12,"A "],["practice",12,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",6,"A "],["practice",6,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",24,"A "],["practice",6,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",12,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",6,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "],["practice",18,"A "]],"hovertemplate":"badge_det=practice<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"practice","marker":{"color":"#19d3f3","pattern":{"shape":""}},"name":"practice","offsetgroup":"practice","orientation":"v","showlegend":true,"textposition":"auto","x":["3","3","3","3","3","3","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","2cem","6","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","6","6","6","6","6","6","6","6","6","6","6","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","5","5","5","5","5","5","5","5","5","5","5","5","5","5","5","5","5","5","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","1cemrm","0cemrupm","0cemrupm","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","4cem","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","3cem","1cemrupm","3cem","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","2cp","2cp","3cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","7","10","7","7","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","9","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","10","12","12","12","12","12","13","12","12","12","12","13","12","12","14","14","14","14","14","14","14","14","14","14","14","14","14","14","13","13","14","14","14","14","14","14","14","14","14","14","13","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","12","8","8","8","8","8","8","8","8","8","8","8","8","13","8","8","8","8","8","8"],"xaxis":"x","y":[6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0,6.0],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",14,"B "],["community",7,"B "],["community",7,"B "],["community",7,"B "],["community",7,"B "],["community",7,"B "],["community",7,"B "],["community",7,"B "]],"hovertemplate":"badge_det=community_practice_makeup<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"community_practice_makeup","marker":{"color":"#FF6692","pattern":{"shape":""}},"name":"community_practice_makeup","offsetgroup":"community_practice_makeup","orientation":"v","showlegend":true,"textposition":"auto","x":["0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","0cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm","1cemrupm"],"xaxis":"x","y":[0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571,0.8571428571428571],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["explore",2,"A-"],["explore",2,"A-"],["explore",4,"A-"],["explore",4,"A-"],["explore",4,"A-"],["explore",4,"A-"],["explore",4,"A "],["explore",4,"A "],["explore",4,"A "],["explore",2,"A "],["explore",2,"A "],["explore",4,"A "],["explore",2,"A "],["explore",2,"A "],["explore",6,"A "],["explore",6,"A "],["explore",6,"A "],["explore",6,"A "],["explore",6,"A "],["explore",6,"A "]],"hovertemplate":"badge_det=explore<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"explore","marker":{"color":"#B6E880","pattern":{"shape":""}},"name":"explore","offsetgroup":"explore","orientation":"v","showlegend":true,"textposition":"auto","x":["5cp","5cp","9","9","9","9","12","12","12","14","14","12","13","13","8","8","8","8","8","8"],"xaxis":"x","y":[9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0,9.0],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["build",1,"B-"],["build",1,"A-"],["build",1,"A "],["build",1,"A "],["build",3,"A "],["build",3,"A "],["build",3,"A "],["build",2,"A "],["build",2,"A "]],"hovertemplate":"badge_det=build<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"build","marker":{"color":"#FF97FF","pattern":{"shape":""}},"name":"build","offsetgroup":"build","orientation":"v","showlegend":true,"textposition":"auto","x":["4","10","12","4cp","11","11","11","13","13"],"xaxis":"x","y":[36.0,36.0,36.0,36.0,36.0,36.0,36.0,36.0,36.0],"yaxis":"y","type":"bar"},{"alignmentgroup":"True","customdata":[["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"D+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"C+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"B+"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A-"],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "],["community",10,"A "]],"hovertemplate":"badge_det=community_plus<br>example=%{x}<br>influence=%{y}<br>badge=%{customdata[0]}<br>count=%{customdata[1]}<br>letter=%{customdata[2]}<extra></extra>","legendgroup":"community_plus","marker":{"color":"#FECB52","pattern":{"shape":""}},"name":"community_plus","offsetgroup":"community_plus","orientation":"v","showlegend":true,"textposition":"auto","x":["0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","0cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","1cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","3cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","2cp","3cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","5cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp","4cp"],"xaxis":"x","y":[1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8,1.8],"yaxis":"y","type":"bar"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmapgl":[{"type":"heatmapgl","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"xaxis":{"anchor":"y","domain":[0.0,1.0],"title":{"text":"example"}},"yaxis":{"anchor":"x","domain":[0.0,1.0],"title":{"text":"grade"},"tickmode":"array","tickvals":[48,102,156,210,30,84,138,192,66,120,174],"ticktext":["D ","C ","B ","A ","D-","C-","B-","A-","D+","C+","B+"]},"legend":{"title":{"text":"badge_det"},"tracegroupgap":0},"margin":{"t":60},"barmode":"relative","height":700},                        {"responsive": true}                    )                };                            </script>        </div>



  
    
    

    
