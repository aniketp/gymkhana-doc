---
title: Nomi-Models 
tags:
keywords:
last_updated:
summary: "This theme primarily uses pages. You need to make sure your pages have the appropriate frontmatter. One frontmatter tag your users might find helpful is the summary tag. This functions similar in purpose to the shortdesc element in DITA."
sidebar: mydoc_sidebar
permalink: mydoc_pages.html
folder: mydoc
---



## Club
Club model is the template of all clubs. The objects represent respective clubs in every council.
**Note:** As an example object, we'll use *Programming Club*

```
 class Club(models.Model):
     club_name = models.CharField(max_length=100, null=True)
     club_parent = models.ForeignKey('self', null=True, blank=True)
     club_members = models.ManyToManyField(User, blank=True)
 
     def __str__(self):
         return self.club_name
```
### Properties
`club_name` : A character field for the name of the club.
 
 e.g : Programming Club

`club_parent` : The Club under which the target club is present
 
 e.g :  Club -> Programing Club, Parent -> SnT Council
 
`club_members` : Linked to User class, list of all club members
 
### Methods
```
 def __str__(self):
          return self.club_name
```
A method to return the name of club when the club object is created.


## Post
Post Model is the template for all the posts within every club.

```
 class Post(models.Model):
     post_name = models.CharField(max_length=500, null=True)
     club = models.ForeignKey(Club, on_delete=models.CASCADE, null=True, blank=True)
     parent = models.ForeignKey('self', on_delete=models.CASCADE, null=True, blank=True)
     post_holders = models.ManyToManyField(User, blank=True)
     post_approvals = models.ManyToManyField('self', related_name='approvals', symmetrical=False, blank=True)
     status = models.CharField(max_length=50, choices=POST_STATUS, default='Post created')
     perms = models.CharField(max_length=200, choices=POST_PERMS, default='normal')
 
     def __str__(self):
         return self.post_name
 
     def remove_holders(self):
         for holder in self.post_holders.all():
             history = PostHistory.objects.get(post=self, user=holder)
             history.end = datetime.now()
             history.save()
 
         self.post_holders.clear()
         return self.post_holders

```

### Properties
`post_name` : Name of the Post , e.g Secretary

`club` : ForeignKey to Club model. Every Post has a Club. And a Club can have multiple Posts.
 
`parent` : The Post which releases nomination for the target Post. e.g Coordinator is the parent of Secretary.
  
`post_holders` : Linked to User class, List of all Post holders.

`post_approvals` : ManytoManyField to self. It is linked to all the posts within the club who are responsible for
 approving the post. Suppose you are a Secretary and want to create a Post (say Freshers) under yourself. Then the
 Coordinators and the Gen-Sec are present in `post_approvals` column. 
 
 `status` : the current status of the Post. Following are the possible options. `choices=POST_STATUS` 

```
 POST_STATUS = (
         ('Post created', 'Post created'),
         ('Post approved', 'Post approved'),
         ('Post rejected', 'Post rejected'),
         ('Post on work', 'Post on work'),
 )
```

`perms` : The permissions of a post. It is differentiated in three possible `choices=POST_PERMS`. 
```
 POST_PERMS = (
     ("normal", "normal"),                                \\ Normal Post Holder  e.g Secretary
     ("can approve the post", "can approve the post"),    \\ Child Post Creator, e.g Coordinator
     ("can approve post and send nominations to users",   \\ The topmost Post,   e.g SnT Gen-Sec
     "can approve post and send nominations to users"),
 )
```
<hr>

### Methods
 `remove_holder()` removes the current `post_holder` from his post and adds the information in the `Post History`.
```
 def remove_holders(self):
      for holder in self.post_holders.all():
          history = PostHistory.objects.get(post=self, user=holder)   
          history.end = datetime.now()                    \\ Adding the Post in PostHistory Model
          history.save()

      self.post_holders.clear()                           \\ Removing the current User
      return self.post_holders                            \\ Returning the remaining list of holders
```



## Post History
PostHistory Model contains the history of a User's Previous and Current Posts.

```
 class PostHistory(models.Model):
     post = models.ForeignKey(Post, on_delete=models.CASCADE, null=True)
     user = models.ForeignKey(User, on_delete=models.CASCADE, null=True)
     start = models.DateField(auto_now_add=True)
     end = models.DateField(null=True, blank=True, editable=True)
```

### Properties
`post` : Linked to the Post for which the history is stored

`user` : Linked to the User whose Post History is being stored

### Date-Field
`start` : The start date for the Post,
          `auto_now_add=True` gets automatically added once a user is assigned a post.
          
`end` : The end date for a Post, `editable=True`: In case the User is continuing for a Post, the end date can be manually 
extended.
          

## Nomination
Nomination Model is the template for every Nomination that is/was released. This class is the heart of the Portal's API.

```
 class Nomination(models.Model):
     name = models.CharField(max_length=200)
     description = models.TextField(max_length=20000, null=True, blank=True)
     brief_desc = models.TextField(max_length=300, null=True, blank=True)
     nomi_post = models.ForeignKey(Post, null=True)
     nomi_form = models.OneToOneField('forms.Questionnaire', null=True, blank=True)
 
     status = models.CharField(max_length=50, choices=STATUS, default='Nomination created')
     nomi_approvals = models.ManyToManyField(Post, related_name='nomi_approvals', symmetrical=False, blank=True)
     club_search = models.ManyToManyField(Post, related_name='all_clubs', symmetrical=False, blank=True)
 
     opening_date = models.DateField(null=True, blank=True)
     closing_date = models.DateField(null=True, blank=True, editable=True)
 
     year_choice = models.CharField(max_length=100, choices=YEAR_1, null=True)
     hall_choice = models.CharField(max_length=100, choices=HALL_1, null=True)
     dept_choice = models.CharField(max_length=100, choices=DEPT_1, null=True)
 
     def __str__(self):
         return self.name
 
     def append(self):
         selected = NominationInstance.objects.filter(nomination=self, status='Accepted')
         self.status = 'Work done'
         self.save()
         for each in selected:
             PostHistory.objects.create(post=self.nomi_post, user=each.user)
             self.nomi_post.post_holders.add(each.user)
 
         return self.nomi_post.post_holders
 
     def replace(self):
         for holder in self.nomi_post.post_holders.all():
             history = PostHistory.objects.get(post=self.nomi_post, user=holder)
             history.end = datetime.now()
             history.save()
 
         self.nomi_post.post_holders.clear()
         self.append()
         return self.nomi_post.post_holders
 
     def open_to_users(self):
         self.status = 'Nomination out'
         self.opening_date = datetime.now()
         self.save()
         return self.status
```

### Properties
`name` : Name of the Nomination

`description` : Description about the Nomination

`brief_desc` : A brief description about the Nomination that is published in the index page

`nomi_post` : Name of the Post for which the Nomination is being released.

`nomi_form` : The Question Form for the Nomination, which the Users are supposed to fill up while applying for the Post.

### Status
`status` : The Status of the Nomination. Following are the possible options, `choices=STATUS`
```
 STATUS = (
         ('Nomination created', 'Nomination created'),
         ('Nomination out', 'Nomination out'),
         ('Interview period', 'Interview period'),
         ('Result compiled', 'Result compiled'),
         ('Work done', 'Work done')
 )
```

`nomi_approvals` : Same as `post_approvals`, linked to all the Users who can approve the Nomination

### Date-Fields
`opening_date` : The date when the Nomination is released, It is automatically assigned when the Gen-Sec approves a 
Nomination, that is, when it is visible for the Users to apply.

`closing_date` : The closing date of the Nomination. `editable=True`: The deadline can be extended, which is usually the
 case most of the time.
 
### Choices
`year_choice` : To specify the batch for whom the Nomination is being released.

`hall_choice` : To specify the hall. Probably will not be used, although it is included in case Hall specific
  nominations are ever released.

`dept_choice` : To specify the department for which the Nomination is being released.
  
### Methods

`append()` method assigns the respective Posts to the selected Users after Gen-Sec approves the result.

```
 def append(self):
     selected = NominationInstance.objects.filter(nomination=self, status='Accepted')
     self.status = 'Work done'                
     self.save()
     for each in selected:            // This loop adds each selected user to the post_holders list
         PostHistory.objects.create(post=self.nomi_post, user=each.user)
         self.nomi_post.post_holders.add(each.user)

     return self.nomi_post.post_holders            // Return final list of users
```

`replace()` method clears all the Post Holders and adds the Post to each individual's Post History.

```
 def replace(self):
     for holder in self.nomi_post.post_holders.all():
         history = PostHistory.objects.get(post=self.nomi_post, user=holder)
         history.end = datetime.now()       
         history.save()          // Saves the post in history

     self.nomi_post.post_holders.clear()
     self.append()               // Clears and saves an empty list
     
     return self.nomi_post.post_holders
```

`open_to_users()` method releases the Nomination for the users when the Gen-Sec approves the Nomination. The status is 
changed to `Nomination Out` and opening date is assigned the current date.

```
 def open_to_users(self):
     self.status = 'Nomination out'
     self.opening_date = datetime.now()
     self.save()
     return self.status
```


## Nomination Instance
The Nomination instance model is the template for keeping the record of each applicant's information about the Nomination.
When a User applies for a Nomination, a Nomination Instance is created for that particular User, linking him to the Nomination.

### Properties
`nomination` : ForeignKey to the Nomination class. The name of the nomination for which the instance is created.
 
`user` : The name of the user who has applied for the Post.

### Choices
`status` : The status of the instance, `choices=NOMI_STATUS`. Either Accepted or Rejected.
```
 NOMI_STATUS = (
         ('Accepted', 'Accepted'),
         ('Rejected', 'Rejected'),
 )
```

`interview_status` : The interview status of the instance, `choices=INTERVIEW_STATUS`.
```
 INTERVIEW_STATUS = (
     ('Interview Not Done', 'Interview Not Done'),
     ('Interview Done', 'Interview Done'),
 )
```

### Forms
`comments` : The Text-Area for adding comments about the User during Interview Period.

`filled_form` : The answer form User filled pre-interview. The Interview panel can view the answers .


## User Profile

```
 class UserProfile(models.Model):
     user = models.OneToOneField(User, on_delete=models.CASCADE)
     name = models.CharField(max_length=40, blank=True)
     roll_no = models.IntegerField(null=True)
     year = models.CharField(max_length=4, choices=YEAR, default='Y16')
     programme = models.CharField(max_length=7, choices=PROGRAMME, default='B.Tech')
     department = models.CharField(max_length=200, choices=DEPT, default='AE')
     hall = models.CharField(max_length=10, choices=HALL, default=1)
     room_no = models.CharField(max_length=10, null=True, blank=True)
     contact = models.CharField(max_length=10, null=True, blank=True)
 
     def __str__(self):
         return str(self.name)
```
### Choices
`year` : Batch of the User, e.g Y16

```
 YEAR = (
     ('Y16', 'Y16'),
     ('Y15', 'Y15'),
     ('Y14', 'Y14'),
     ('Y13', 'Y13'),
     ('Y12', 'Y12'),
     ('Y11', 'Y11'),
 )
```

`programme` : Either B.S or B.tech

```
 PROGRAMME = (
         ('B.Tech', 'B.Tech'),
         ('B.S', 'B.S'),
 )
```

`department` : The dept of the User

```
 DEPT = (
     ('Aerospace Engineering', 'AE'),
     ('Biological Sciences & Engineering', 'BSBE'),
     ('Chemical Engineering', 'CHE'),
     ('Civil Engineering', 'CE'),
     ('Computer Science & Engineering', 'CSE'),
     ('Electrical Engineering', 'EE'),
     ('Materials Science & Engineering', 'MSE'),
     ('Mechanical Engineering', 'ME'),
     ('Industrial & Management Engineering', 'IME'),
     ('Chemistry', 'CHM'),
     ('Mathematics & Scientific Computing', 'MTH'),
     ('Physics', 'PHY'),
     ('Earth Sciences', 'ES')
 )
```