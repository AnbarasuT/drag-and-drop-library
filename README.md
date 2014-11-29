drag-and-drop-library
=====================

/* ----DragNDrop.js 0.8
 
 */
window.DragNDrop = function(Container, Details){
    this.Container = Container;
    this.Details = Details ? Details:{};
    this.SourceElem;
    this.DragElements;
    this.DestinationElems;
    this.isTouched = false;
    this.DownX = 0;
    this.DownY = 0;
    this.TouchedIndex = 0;
    this.FirstInStackClass = "firstInStack";
    this.BackInStackClass = "waitingToDrag";
    this.ElementsClass = "draggableEle";
    this.DropClass = "droppedEle";
    if (this.Container) {
        this.Manifest = this.Details.Manifest ? this.Details.Manifest:null;
        this.Options = this.Details.Manifest.Options ? this.Details.Manifest.Options:{};
    }
    this.CheckTimer;
    this.AutoRevert = this.Options.AutoRevert ? this.Options.AutoRevert:true;
    this.IsBusy = false;
    this.AniTime = 300;
    this.DragItem = false;
    if (this.Options.BoundariesId) this.Boundary = $("#"+this.Options.BoundariesId);
    if (this.Manifest && this.Container) {
        this.init();
    }
};
DragNDrop.prototype = {
    init:function(){
        try {
	    this.IsBusy = true;
            var drgNdrp = this;
            var allContainers = this.Manifest.Destinations.slice(0);
            allContainers.push(this.Manifest.Source);
            this.SourceElem = $("#"+this.Manifest.Source.ContainerId);
	    this.SourceElem.html("");
            for (var cont=0; cont<allContainers.length; cont++) {
                var container = $("#"+allContainers[cont].ContainerId);
                var srcPos = container.css("position").toString().toLowerCase();
                if (srcPos.indexOf("fixed") < 0 && srcPos.indexOf("relative") < 0 && srcPos.indexOf("absolute") < 0)
                    container.css({"position":"relative"});
                container.css({"vertical-align":"top"});
            }
            if (this.Options.Random)
            for(var i=0; i<this.Manifest.Source.Items.length; i++){
                var a = this.Manifest.Source.Items[i];
                var randomIndex = Math.round(Math.random()*(this.Manifest.Source.Items.length-1));
                var b = this.Manifest.Source.Items[randomIndex];
                this.Manifest.Source.Items[i] = b;
                this.Manifest.Source.Items[randomIndex] = a;
            }
            this.DragElements = [];
            for(var i=0; i<this.Manifest.Source.Items.length; i++){
                var srcItem = this.Manifest.Source.Items[i];
                var item;
                if (srcItem.OuterHTML) item = $(srcItem.OuterHTML);
                else if (srcItem.InnerHTML) item = $("<div>"+srcItem.InnerHTML+"</div>");
                else item = $("<div />")
		item.css({"visibility":"hidden"});
                if (this.Manifest.Source.ItemClass) item.addClass(this.Manifest.Source.ItemClass);
                if (this.Manifest.Source.FirstItemClass) this.FirstInStackClass = this.Manifest.Source.FirstItemClass;
                if (srcItem.Id) item.attr("id",srcItem.Id);
                item.addClass(this.ElementsClass);
                if (i==this.Manifest.Source.Items.length-1) item.addClass(this.FirstInStackClass);
                else item.addClass(this.BackInStackClass);
                this.SourceElem.append(item);
                var elemDetail = {
                    Element:item,
                    OrgX:0,
                    OrgY:0,
                    Value:srcItem.Value
                }
                this.DragElements.push(elemDetail);
                item.attr("elementValue",srcItem.Value);
                if (this.Options.Arrangement){
                    var maxL = this.SourceElem.width()-item.width();
                    var maxT = this.SourceElem.height()-item.height();
                    if(this.Options.Arrangement.InSource.toString().indexOf("stack") >= 0){
                        var l = Math.round(Math.random() * maxL);
                        var t = Math.round(Math.random() * maxT);
                        item.css({"position":"absolute","left":l+"px","top":t+"px"});
                    }
		    else item.css({"display":"inline-block"});
                }
            }
            for (var index=this.DragElements.length-1; index>=0; index--) {
		
                var ele = this.DragElements[index].Element;
                if (this.Options.Arrangement)
                    if(this.Options.Arrangement.InSource.toString().indexOf("order") >= 0)
                        ele.css({"position":"absolute","left":(ele.position().left)+7+"px","top":ele.position().top+"px","z-index":index+100});
                    else ele.css({"z-index":index+100});
                this.DragElements[index].OrgX = ele.position().left;
                this.DragElements[index].OrgY = ele.position().top;
                ele.bind('touchstart mousedown',function(event){ drgNdrp.onItemTouchStart(event);});
            }
            $(window).bind('touchmove mousemove',function(event){ drgNdrp.onItemTouchMove(event);});
            $(window).bind('touchend mouseup',function(event){ drgNdrp.onItemTouchEnd(event);});
            this.DestinationElems = [];
            for(var d=0; d<this.Manifest.Destinations.length; d++){
                var destDtails = {
                    Element:$("#"+this.Manifest.Destinations[d].ContainerId),
                    Value:this.Manifest.Destinations[d].Value
                }
                this.DestinationElems.push(destDtails);
            }
	    for (var index=this.DragElements.length-1; index>=0; index--) {
		this.DragElements[index].Element.css({"visibility":"visible"});
	    }
        } catch(e) { }
	this.IsBusy = false;
    },
    onItemTouchStart:function(event){
        event.preventDefault();
        var drgNdrp = this;
        if (!this.IsBusy) {
            var element = $(event.target);
            element = $(event.target).parents("."+this.ElementsClass);
            for (var ele=0; ele<this.DragElements.length; ele++) {
                var zInd = this.DragElements[ele].Element.css("z-index");
                if(this.DragElements[ele].Element[0] == element[0]){
                    this.DragElements[ele].Element.removeClass(this.DropClass);
                    this.isTouched = true;
                    this.TouchedIndex = ele;
                    var parentOffset = element.parent().offset();
                    try{
		    this.DownX = event.originalEvent.touches[0].pageX - parentOffset.left - element.position().left;
		    this.DownY = event.originalEvent.touches[0].pageY - parentOffset.top - element.position().top;
		    }catch(e){
		    this.DownX = event.pageX - parentOffset.left - element.position().left;
		    this.DownY = event.pageY - parentOffset.top - element.position().top;
		    }
                    this.DragElements[ele].Element.css({"z-index":(1000 + Number(this.DragElements.length-1)),"position":"absolute"});
                    this.DragElements[ele].Element.addClass(this.FirstInStackClass);
                    this.DragElements[ele].Element.removeClass(this.BackInStackClass);
                    if (this.Boundary)
                        this.changeContainer(this.DragElements[ele].Element,this.Boundary);
                        try {
			    this.onMove(event.originalEvent.touches[0].pageX, event.originalEvent.touches[0].pageY);
			} catch(e) {
			    this.onMove(event.pageX, event.pageY);
			}
                }
                else{
                    this.DragElements[ele].Element.css({"z-index":zInd-1});
                    this.DragElements[ele].Element.addClass(this.BackInStackClass);
                    this.DragElements[ele].Element.removeClass(this.FirstInStackClass);
                }
            }
        }
    },
    onCheckChange:function(){
        if (this.TouchedX || this.TouchedY) {
            this.onMove(this.TouchedX, this.TouchedY);
        }
    },
    onItemTouchMove:function(event){
        var touchX;
        var touchY;
        try {
            touchX = event.originalEvent.touches[0].pageX;
            touchY = event.originalEvent.touches[0].pageY
        } catch(e) {
            touchX = event.pageX;
            touchY = event.pageY;
        }
        this.TouchedX = touchX;
        this.TouchedY = touchY;
        this.onMove(touchX, touchY);
    },
    onMove:function(pageX,pageY){
        if (this.isTouched && !this.IsBusy) {
            var element = this.DragElements[this.TouchedIndex].Element;
            var parentOffset = element.parent().offset();
            var l = pageX - parentOffset.left - this.DownX;
            var t = pageY - parentOffset.top - this.DownY;
            var a = true;
            if (this.Boundary)
                a = (l > 0 && l < this.Boundary.width()-element.width() && t > 0 && t < this.Boundary.height()-element.height());
            if (a) element.css({"left":l+"px", "top":t+"px"});
        }
    },
    onItemTouchEnd:function(event){
        if (this.isTouched) {
            this.isTouched = false;
            if(event.type == "touchend"){
		var touch = event.originalEvent.touches[0] || event.originalEvent.changedTouches[0];
		var touchX = touch.pageX;
		var touchY = touch.pageY;
	    }else{
		var touchX = event.pageX;
		var touchY = event.pageY;
	    }
            var drgNdrp = this;
            drgNdrp.IsBusy = true;
	    window.drgNDrop = drgNdrp;
            var droppedBox = -1;
            var elementObj = this.DragElements[this.TouchedIndex];
            var parentOffset = elementObj.Element.parent().offset();
            for (var d=0; d<this.DestinationElems.length; d++) {
                var destEl = this.DestinationElems[d].Element;
                if(destEl.offset().left < touchX && destEl.offset().left + destEl.width() > touchX &&
                   destEl.offset().top < touchY && destEl.offset().top + destEl[0].scrollHeight > touchY){
                    droppedBox = d;
                    drgNdrp.IsBusy = true;
                    this.changeContainer(elementObj.Element,destEl,parentOffset, true);
                    this.onChangeinDestination(elementObj.Element, destEl, true, false);
                    elementObj.Element.addClass(this.DropClass);
		    window.drgNDrop.onItemChange();
		    elementObj.Element.removeClass("slider");
		    resetEnable();
                    break;
                }
            }
            if (droppedBox == -1 && this.AutoRevert) {
                this.changeContainer(elementObj.Element, this.SourceElem, parentOffset);
		    var isVariationMore = Math.abs(elementObj.Element.position().left-elementObj.OrgX) > 20 || Math.abs(elementObj.Element.position().top-elementObj.OrgY) > 20;
		if (isVariationMore) {
		    elementObj.Element.animate({"left":elementObj.OrgX+"px","top":elementObj.OrgY+"px"},{complete:function(){
		    drgNdrp.onItemChange();
		    drgNdrp.triggerChange();
		    drgNdrp.setDroppableHeight();
                    },duration:this.AniTime});
		}
		else {
		    elementObj.Element.css({"left":elementObj.OrgX+"px","top":elementObj.OrgY+"px"});
		    drgNdrp.onItemChange();
		    drgNdrp.triggerChange();
		    drgNdrp.setDroppableHeight();
		}
            }
            for(var d=0; d<this.DestinationElems.length; d++)
                for (var e=0; e<this.DestinationElems[d].Element.children().length; e++){
                    this.onChangeinDestination($(this.DestinationElems[d].Element.children()[e]), this.DestinationElems[d].Element, false, true);
                }
	    setTimeout(function(){drgNdrp.IsBusy = false;}, this.AniTime*1.2);
        }
        else this.isTouched = false; 
    },
    onItemChange:function(){},
    triggerChange:function(){},//Here scroll
    onChangeinDestination:function(Element, Destination, checkFinal, isFastMove){
        var drgNdrp = this;
        var lX = Element.position().left;
        var lY = Element.position().top;
        var xPos = (Destination.width() - Element.width()+10)/2;
        var ind = 0;
        for (var i=0; i<Destination.children().length; i++)
            if(Destination.children()[i] == Element[0]){
                ind = i
                break;
            }
        var yPos = (Element.height() * ind);
        Element.removeClass(this.BackInStackClass);
	var aniTime = this.AniTime;
	if (isFastMove) aniTime *= 0.5;
	if (Math.abs(yPos - lY) > 5 || Math.abs(xPos - lX) > 5) {
	    Element.css({"left":xPos+"px","top":yPos+"px"});
	    if (checkFinal) {
		drgNdrp._onAllDropComplete();
		drgNdrp.setDroppableHeight();
		drgNdrp.triggerChange();//Here scroll
	    }
		/*{complete:function(){
		    if (checkFinal) {
			drgNdrp._onAllDropComplete();
			drgNdrp.setDroppableHeight();
			drgNdrp.triggerChange();//Here scroll
		    }
		},duration:aniTime});*/
	}
	else{
	    Element.css({"left":xPos+"px","top":yPos+"px"});
	    if (checkFinal) {
		drgNdrp._onAllDropComplete();
		drgNdrp.setDroppableHeight();
		drgNdrp.triggerChange();//Here scroll
	    }
	}
    },
    setDroppableHeight:function(){//New Function
	for(var i=0; i<this.DestinationElems.length; i++){
	    var droppable = this.DestinationElems[i].Element;
	    var innerHeight = droppable.children().height()*droppable.children().length;
	    var minHeight = droppable.parent().height();
	    var height = innerHeight;
	    if (innerHeight < minHeight) height = minHeight;
	    droppable.css({"height":height+"px"});
	}
    },
    _onAllDropComplete:function(){
        var childrenCount = 0;
        for (var i=0; i<this.DestinationElems.length; i++) {
            childrenCount += this.DestinationElems[i].Element.children().length;
        }
        if (childrenCount == this.DragElements.length) {
            this.validate();
            this.OnAllDropComplete();
        }
    },
    OnAllDropComplete:function(){},
    validate:function(){
        var correct = 0;
        var incorrect = 0;
        for (var i=0; i<this.DestinationElems.length; i++) {
            for (var e=0; e<this.DestinationElems[i].Element.children().length; e++) {
                if($(this.DestinationElems[i].Element.children()[e]).attr("elementvalue") == this.DestinationElems[i].Value) correct++;
                else incorrect++;
            }
        }
        this.Correct = correct;
        this.Incorrect = incorrect;
    },
    changeContainer:function(Element,NewContainer,ParentOffset, checkDropOrder){
        if (!ParentOffset) ParentOffset = Element.parent().offset();
try{        var prevX = ParentOffset.left;
	    var prevY = ParentOffset.top;}catch(e){}
        NewContainer.append(Element);
        ParentOffset = Element.parent().offset();
        var currX = ParentOffset.left;
        var currY = ParentOffset.top;
        var objX = Element.position().left;
        var objY = Element.position().top;
        var l = objX + (prevX - currX);
        var t = objY + (prevY - currY);
        Element.css({"left":l+"px","top":t+"px"});
        if (checkDropOrder){
            var changed = false;
            for (var i=0; i<NewContainer.children().length; i++){
                var sibling = NewContainer.children()[i];
                if (Element[0] != sibling && Element.position().top < $(sibling).position().top){
                    $(sibling).before(Element);
                    this.checkAllElementPos(NewContainer);
                    break;
                }
            }
        }
    },
    checkAllElementPos:function(NewContainer){
        for (var i=0; i<NewContainer.children().length; i++){
            this.IsBusy = true;
            this.onChangeinDestination($(NewContainer.children()[i]), NewContainer, true, true);
        }
    }
};



activity.js
============


var manifest={
    Source:{
	ContainerId:"div_source",
	ItemClass:"dragItem",
	FirstItemClass:"firstInStack",
	Items:[
	    {InnerHTML:"<div id=\"item1\" class=\"innerDiv one\">7 = 7</div>", Value:"1"},
	    {InnerHTML:"<div id=\"item2\" class=\"innerDiv two\">8 = 10 &minus; 6</div>", Value:"2"},
	    {InnerHTML:"<div id=\"item3\" class=\"innerDiv two\">7 + 6 = 13</div>", Value:"2"},
	    {InnerHTML:"<div id=\"item4\" class=\"innerDiv one\">3 + 4 = 5 + 2</div>", Value:"1"},
	    {InnerHTML:"<div id=\"item5\" class=\"innerDiv one\">7 &minus; 1 = 6 &minus; 2</div>", Value:"1"},
	    {InnerHTML:"<div id=\"item6\" class=\"innerDiv two\">8 + 3 = 7 + 9</div>", Value:"2"},
	    {InnerHTML:"<div id=\"item7\" class=\"innerDiv one\">7 + 8 = 8 + 7</div>", Value:"1"},
	]
    },
    Destinations:[
	{ContainerId:"droppableArea1", Value:"1"},
	{ContainerId:"droppableArea2", Value:"2"}
    ],
    Options:{
	BoundariesId:"DragActivity",
	AutoRevert:true,
	FixOnDrop:true,
	Random:true,
	Arrangement:{
	    InSource:"order",
	    InDestination:"order"
	}
    }
}


dragActivity = new DragNDrop(dragContainer,{Manifest:manifest});


