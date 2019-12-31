```
var dsViewVue = Vue.extend({
    template: '<div>' + resp + '</div>',
    i18n,
    components:{
        editor
    },
    data: function(){
        return{
            curWidget: _this.curWidget
        }
    }
});
var dsView = new dsViewVue().$mount();
$('#dsView').html(dsView.$el);
```
