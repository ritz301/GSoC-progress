## Table of Contents

* [Basic Information](#basic-information)
* [Personal Background](#personal-background)
* [The Project](#the-project)
* [The Plan & Prototype](#the-plan--prototype)
* [Timeline (Tentative)](#timeline-tentative)
* [Notes](#notes)
* [References](#references)

---
# About Me   

## Basic Information

**Name:** Ritvik Raj<br/>
**University:** [Indian Institute of Technology, Kharagpur](http://en.wikipedia.org/wiki/Indian_Institute_of_Technology_Kharagpur)<br/>
**Department:** Computer science & engineering<br/>
**Email:** ritvik.raj.satya@gmail.com<br/>
**IRC:** ritz301<br/>
**Github:** [ritz301](https://www.github.com/ritz301)<br/>
​**Blog:** http://ritzz301.blogspot.in/<br/>
**Timezone:** IST (UTC +5:30)<br/>

## Personal Background

My name is Ritvik Raj, a second year undergraduate student at IIT Kharagpur, India. I am pursuing a degree in Computer Science.

I use Ubuntu 14.04 LTS on my workstation and Sublime text 3 as my primary text editor. I've been coding in C, C++ & python for over 2 years now and in java for under a year and am proficient in all four of them. 

Since I started programming as a web developer, I used Django all the time which made me familiar with Python in the beginning. 

I'm very much interested in web development and work as a Sub Head , Web team of our college cultural fest. Therefore, I have a good idea of basic elements of web designing, namely:
* HTML5, CSS, CSS3, JSON, REST, MySQL, PHP, Django, JS, jQuery, Ajax, Bootstrap & Materialize frameworks. 

I know how to use Git and Github. If I am stuck, I go to Google and always come back with a solution.

# The Project  
 
## Motivation	   

I'm fascinated by the idea about my work being beneficial to other people in the open source community. The reason why I chose to work on this project is because it is well aligned with my skills and interests. I started programming as a web developer and have developed numerous applications ever since. Besides, I have previously developed similar management systems which can be seen on my [github handle] (https://github.com/ritz301). 
   
The software I'm going to develop will make the process of adding annotations along with additional notes really simple and flexible. While doing so, my experience with Shotgun annotations will complement my proficiency in designing skills. Also, the skills and experience I'm going to acquire during the process is unparalleled.
       
Furthermore, the software developed is not only limited to peragro and can be applied in other fields as well. The Django project is a generic  application for project management. The scope and usage of this is beyond Peragro. This fact sways me even more.

## The Plan & Prototype

Besides software documentation and testing, my work can be divided into three phases:

#### Phase 1: Designing the User Interface
The structure of the UI will have two major sections:
* **Displayed Asset annotation preview**

 This section will enable the user to draw strokes over a chosen background image which will be in a    *OneToOne-relationship* with the corresponding comment. The drawn strokes are later saved as a vector graphic serialized to JSON. A sample preview of the created annotation is shown:

 ![Asset annotation sample preview](http://ritz301.net23.net/test/anno1.jpg)

 As you can infer, the user is allowed to draw strokes with a particular color and stroke size. Additionally, the user can select a background image to be saved with that drawn stroke. I have created a [**live demo**](http://ritz301.net23.net/test/test.html) of the mentioned as discussed in IRC.

 Further, there will be options which will specify the image format, size and type to be displayed in the assets preview. I can think of many more features to be added here. (Putting a text on the image, drawing a straight line, rectangle, ellipse etc.). 

 In the live demo, I have also created options for *serializing* and *deserializing* the drawn annotation for now. It serializes the drawn strokes to JSON and displays the result and vice-versa. This will not be a part of the final UI (rather handled internally).

* **Comments section**

 This will be a forum-like-structure where each comment is linked to an annotation which will be displayed on the annotation section when the user selects it. Above this, there will be a set actions user can perform before/after selecting a comment ( `Add`, `Edit`, `Update`, `Delete` etc. ). Plus, the user will be allowed to arrange the comments in any desired order (spatial, chronological or any custom order). Many more features can added here taking inspiration from Shotgun Software. Finally, this section will have a `Submit` button which will first validate and then send an `Ajax POST` request to the API endpoint `.../assets/` to make the changes in the Database. 

Every element of the UI will be a REST element.
 
#### Phase 2: Designing the REST API with Django

This is the core part of the project. The serialized vector graphic from the UI will be stored with an asset ID along with the relevant information (Title + JSON + Image + Comment) via Django models. A simple definition of the model class is shown: 

```python
class Asset(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    comments = models.TextField()
    json = models.TextField()
    pic = models.ImageField("Image", upload_to="images/") 
    owner = models.ForeignKey('auth.User', related_name='assets')   
    class Meta:
        ordering = ('created',)
    def save(self, *args, **kwargs):
	    ...
	    super(Asset, self).save(*args, **kwargs)
```
The first thing we need to get started on our Web API is to provide a way of serializing and deserializing the asset instances into representations such as `json`. We can do this by simply adding a serializer class as shown:

```python
class AssetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Asset
        fields = ('id', 'title', 'comments', 'json', 'pic')
```

Next major milestone would be creating various much needed API endpoints to display the assets to the user. Since we want the drawing to be recreated by any client with the least amount of data transfer or storage, this would require URLs with particular endpoints (to specify transcode type, image size, image quality etc.). For example, to display asset image of size `256*256`, format `png` and type `interlaced` associated with assetID 2, the url would go something like:

    .../assets/2/image__png/256/256/interlaced

This will display only the image part of the asset. To display the complete asset annotation, we can have a different API endpoint something like this:

    .../assets/2/cimage__png/256/256/interlaced

Next comes the API endpoints required for handling `GET` and `POST` requests on assets (`.../assets/`). This can be handled by adding the following function-based view:

```python
@csrf_exempt
def asset_list(request):
    """
    List all assets, or create a new asset.
    """
    if request.method == 'GET':
        assets = Asset.objects.all()
        serializer = AssetSerializer(assets, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = AssetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)
```    
Now for retrieving, updating or deleting a particular asset we require separate endpoints (like `.../assets/1`) handling the `GET`, `PUT` and `DELETE` requests. This can be handled by adding the following function-based view:

```python
@csrf_exempt
def asset_detail(request, pk):
    """
    Retrieve, update or delete an asset instance.
    """
    try:
        asset = Asset.objects.get(pk=pk)
    except Asset.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = AssetSerializer(asset)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = AssetSerializer(asset, data=request.data)
        if serializer.is_valid():
            asset.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        asset.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
``` 
I have created a [**live demo**](http://45.55.91.182:8080/) of the above mentioned as discussed in IRC using the default Django User interface. You can login as:    
`Username : user1`    
`Password : password123`     
to see the specific functions user is authorised to perform. It's sufficient to use the default `Django-admin` interface for the admin as discussed in IRC. So there is nothing much to do on admin part here.

#### Phase 3: Integration of the UI and the API

The final work. It mainly involves integrating the Django API User functionalities created in `Phase 2` with the User Interface created in `Phase 1`. This can be done with the use of `Django’s template system` to separate the design from Python by creating a template that the view can use. 

Templates can be implemented by adding a function-based view and rendering the context (asset details) with it. 
Below is a small script for displaying the image for the URL endpoint defined by the regex  `r'^assets/(?P<pk>[0-9]+)/cimage__(?P<type>(jpe?g|png|gif|bmp)+)/(?P<h>[0-9]+)/(?P<w>[0-9]+)$'` using templates: 


```python
def cmod_image(request, pk=1, type="png", h="256", w="256"):
    try:
        asset = Asset.objects.get(pk=pk)
    except Asset.DoesNotExist:
        return HttpResponse("Asset ID " + pk + " does not exist", status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
    	x =  asset.pic
    	im = Image.open(x.path)
    	infile = x.path
    	f, e = os.path.splitext(infile)
        outfile = f + "." + type
        im.save(outfile, type)
        template = loader.get_template('assets/casset.html')
        context = {
            'img': x.name.split(".")[0] + "." + type,
            'json': asset.json,
            'h': h,
            'w': w,
            'comments': asset.comments,
            'type': type,
        }
        return HttpResponse(template.render(context, request))
```

The context variables (say `img`) are rendered into templates (`assets/casset.html`) and can be accessed using `{{variable_name}}` inside (here `{{img}}`) it. 

Similar process can be used to integrate other views with the Django Rest API.


#Timeline (Tentative)

##### Community Bonding Period (April 22 - May 22)

**Goal**: Community Bonding

* The principle focus in this period would be studying in detail the Django Project and other projects within Peragro. It'll also involve making notes and interacting with different contributors.
* If possible, I would also start coding in this phase only, so that I get a head-start.

##### Week 1 - Week 2 (23 May - 5 June)

**Goal**: Creating the User Interface Skeletal

* I'll create the general UI skeletal. It will have a basic design and will cover most of the front end functionalities as described in `Phase 1`.
* I'll out from 29 May to 5 June on a vacation as informed in IRC. I'll cover the vacations job by the end of this period itself. 

##### Week 3 - Week 4 (6 June - 19 June)

**Goal**: Completing the User Interface design

* This will basically involve putting all elements together in an elegant way.
* It will probably involve the use of Materialize/Bootstrap CSS frameworks. The work will be hosted on github by the end of this week.

##### Week 5 (20 June - 26 June)

**Goal**: Setup the Django model & Database

* Backbone of the project.
* This would involve setting up model and serializer classes in Peragro Django Project. 

##### Mid term Evaluation

* A proper user interface with all front end functions working nicely.
* API Backbone setup.

##### Week 6 (27 June - 3 July)

**Goal**: Adding & setting up relevant API endpoints

* In this period I'll be writing routing schemes for the root and the API as described in `Phase 2`. 
* Adding function-based views for handling HTML requests

##### Week 7 - Week 8  (4 July - 17 July)

**Goals**: Adding authentication & permissions, Exception handling   

* Adding user authentication and permissions
* Adding Admin Interface
* The work will be hosted on Github by the end of this week.

##### Week 9 - Week 10  (18 July - 31 July)

**Goal**: Integration of the API & the UI

* REST API and the view will be integrated as described in `Phase 3` 
* Finishing/fixing bugs for existing features so far.

##### Week 11 - Week 13 (1 August - 22 August)

**Goal**: Software documentation & testing

* I will be writing the tests and documentation for every class and function I have implemented.
* Buffer period
* The final software will be hosted on github by the end of this period.

### Post GSoC    

Continue working over the Django project forever. I'll love to see future implementations on it.

My summer vacation starts on 29th April. Hence, I will start coding a month earlier than the GSoC coding period, effectively giving me 4 months to complete the project. Though my classes start in mid July, it won't be an issue as I won't have any tests or exams then.

### Notes

  * I have no major commitments for this summer and I am positive that I will be able to contribute for about 40-50 hours a week for the project. I will try my best to learn as much as I can, during the project. 
  
  * I am very enthusiastic about my work being beneficial to other people in the open source  community. I'll keep on seeing the work done by me and fixing issues if they emerge in future.

### References

* Shotgun annotations:   
  * https://shotgunsoftware.com/ 
  * https://www.youtube.com/watch?v=A2xG7NgodMM&feature=youtu.be&t=1m11s
* Asset annotation preview:    
  http://ramkulkarni.com/blog/record-and-playback-drawing-in-html5-canvas-part-ii/
* Project idea: https://github.com/peragro/peragro/wiki/GSoC-Ideas#annotations
* Source code : https://github.com/peragro/django-project
