<template>
  <div class="page-view cp-page-points  " id="Page-address-select" >
    <title-bar  :left-options="{showBack: true}">
      <span v-show="isShowSearchBox==0">{{pageTitle}}</span>
      <cp-search-box slot="rightContent" @on-show-input="showSearchBox(1)" @on-hide-input="showSearchBox(0)" v-model="keyword" @on-keyup="doSearch" ></cp-search-box>

      <!--<div slot="rightContent">
        <div class="cp-search-box" v-show="isShowSearchBox"><input name="keyword" class="form-control form-control-line" placeholder="请输入关键字查找"  v-model="keyword" @keyup="doSearch" autocomplete="false" ></div>
        <div class="cp-btn-wrapper">
          <button class="cp-btn-search" v-show="isShowSearchBox==0" @click="showSearchBox(1)"><i class="fa fa-search"></i></button>
          <button class="cp-btn-close" v-show="isShowSearchBox" @click="showSearchBox(0)"><i class="fa fa-times"></i></button>
        </div>
      </div>-->
    </title-bar>


    <div class="page-view-main " >
      <cp-scroller :position="{top:'46px'}"  :on-refresh="onRefresh"   :dataList="scrollData" :enableInfinite="false">
        <ul id="J-getAddress" class="cp-list-wrap cp-list-points " :class="{'doSearch':isSearching}"  >
          <li v-for="(item,index) in listDatas" class="cp-item" :class="'cp-type-'+item.address_type"  @click="onSelectAddress(index,0)" v-show="item.is_show">
            <template v-if="item.address_type=='Home'">
              <i class="fa fa-home"></i><h6>家</h6>
            </template>
            <template v-else-if="item.address_type=='Work'">
              <i class="fa fa-suitcase"></i><h6>公司</h6>
            </template>
            <template v-else>
              <i class="fa fa-map-pin"></i>
            </template>
          <b class="name">{{item.addressname}}</b>
          <p class="address">{{item.address}}</p>
          </li>
        </ul>
        <ul id="J-list-mapAddress" class="cp-list-wrap cp-list-points"   >
          <li v-for="(item,index) in smListDatas" class="cp-item cp-type-frommap"   @click="onSelectAddress(index,1)" >
          <i class="fa fa-map-marker"></i>
          <b class="name">{{item.addressname}}</b>
          <p class="address">{{typeof(item.address)=="string" ? item.address : item.district}}</p>
          </li>
        </ul>


       <span slot="loading-text"><spinner type="dots" size="60px"></spinner></span>
       <div class="text-center">
         <div class="cp-nodata-tips" v-show="noData">
           暂时没有数据 ⁽⁽ƪ(ᵕ᷄≀ ̠˘᷅ )ʃ⁾⁾
         </div>
         <spinner type="dots" size="60px" v-show="isLoading"></spinner>
       </div>
       <div class="cp-createAddress-box" @click="goCreateAddress" v-show="isShowCreateBtn">
         <p>没找到您想要的站点？</p>
         <p><i class="fa fa-plus"></i> 创建站点：<b class="cp-keyword"></b></p>
       </div>
      </cp-scroller>

    </div>
    <div style="display:none" id="cp-map-hidden"></div>
  </div>

</template>

<script>
import config from '../configs/index'
import cFuns from '../utils/cFuns'
import CpSearchBox from '../components/CpSearchBox'
import cModel from '../utils/cModel'

export default {
  components: {
    CpSearchBox
  },
  data () {
    return {
      to: this.$route.params.to,
      keyword : '',
      isShowSearchBox:0,
      listDatas :[],
      smListDatas :[],
      noData:0,
      scrollData: {
          noFlag: false //暂无更多数据显示
      },
      enableInfinite:false,
      isLoading:true,
      isSearching:false,
      uid:localStorage.getItem('CP_uid'),
    }
  },
  computed:{
    pageTitle (){
      let titlesArray = {
        "start":"请选择开始地点",
        "end":"请选择结束地点",
        "home":"请选择家地址",
        "work":"请选公司地址",
      }
      return typeof(titlesArray[this.to]!=="undefault") ? titlesArray[this.to] : "请选择地址"
    },
    isShowCreateBtn (){
      return this.to == "start" || this.to=="end" ? true : false;
    }
  },
  methods:{


    /**
     * [showSearchBox 显示或关闭搜索输入]
     */
    showSearchBox (show){
      this.isShowSearchBox = show ? 1 : 0;
      if(show==0){
        this.keyword = '';
        this.getList();
        this.isSearching = false;
      }
    },

    /**
     * [doSearch 进行搜索]
     */
    doSearch (){
      let _this = this;
      if(this.keyword_o !==  this.keyword ){
        this.keyword_o =  this.keyword
        // console.log(this.keyword)

        if(this.keyword==""){
          this.getList();
          this.isSearching = false;
        }else{
          this.isSearching = true;
          let  temp_array = this.listDatas;
          this.listDatas = temp_array.map(function(value,key,arr){

            if(value.addressname && value.addressname.indexOf(_this.keyword)>-1  ){
              value.is_show = 1
            }else{
              value.is_show = 0
            }
            return value;
          })
          this.searchMapAddress();
        }
        // console.log(this.listDatas);
        // this.getList(1)
      }
    },

    /**
     * [getList 显示或关闭搜索输入]
     */
    getList(refresh){
      var nowTimestamp = new Date().getTime();
      var _this = this;
      refresh = refresh || 0;
      if(refresh){
        return _this.loadList();
      }

      //打开本地数据库 查询地址列表
      cModel.myAddress('getAll',{
        orderBy:'listorder',
        success:function(results,server){
          var overTime = window.localStorage.getItem('CP_'+_this.uid+'_addressOverTime'); //上次记录本地数据的时间
          overTime = overTime ? overTime : 0;
          // console.log(overTime)
          //当本地数据为空，或者数据过期时，重新获取
          if(!results || !results.length || ((nowTimestamp - overTime) > 7*24*60*60*1000) ){
            // console.log(overTime)
            _this.loadList();
          }else{
            _this.listDatas = results;
            _this.isLoading = 0;
            _this.listDatas.forEach(function(value,key,arr){
              // console.log(value)
              value.is_show = 1;
              // console.log(value.address_type);
            })
          }

        }
      });


    },

    /**
     * [getCity 通过高德地图定位到当前城市]
     */
    getCity(){
      var _this =this;
      var map = new AMap.Map("cp-map-hidden",{

      });
      map.getCity(function(data) {
        // console.log(data)
          if (data['province'] && typeof data['province'] === 'string') {
            _this.$store.commit('setLocalCity',data);
          }
      });
    },

    /**
     * 通过接口取地址列表数据
     * @param  {function} success [成功回调]
     */
    loadList (success) {
      var _this = this;
      var nowTimestamp = new Date().getTime();

      // console.log(config.urls.checkLogin)
      // alert(1)
      let params = {keyword:this.keyword,page:1};

      _this.isLoading = 1;
      _this.noData = 0;
      _this.$tokenAxios.get(config.urls.getMyAddress,{params:params}).then(res => {
        // console.log(res)

          _this.isLoading = 0;
          if(res.data.code === 0) {
            let data = res.data.data;

            cModel.myAddress('clear');
            window.localStorage.setItem('CP_'+this.uid+'_addressOverTime',nowTimestamp);

            _this.listDatas = data.lists.map(function(value,key,arr){
              value.listorder = key;
              value.address = '';
              let address;
              if(value.address_type!='Home' && value.address_type!='Work'){
                if(value.addressid==arr[0].addressid || value.addressid==arr[1].addressid){
                  return value;
                }
              }
              cModel.myAddress('add',{data:value});
              value.is_show = true;
              return value;
            })


            //通高德地图以地坐标取得地址信息，并写入本地数据库
              AMap.plugin('AMap.Geocoder',function(){
                var geocoder = new AMap.Geocoder({
                     radius: 1000,
                     extensions: "all"
                 });
                 _this.listDatas.forEach(function(value,key,arr){
                   // console.log(value)
                   // console.log(value.address_type)

                   if(value.address_type!='Home' && value.address_type!='Work'){
                     if(value.addressid==arr[0].addressid || value.addressid==arr[1].addressid){
                       return ;
                     }
                   }
                   geocoder.getAddress([value.longtitude,value.latitude], function(status, result) {
                       if (status === 'complete' && result.info === 'OK') {
                         value.listorder = key;
                         value.address = result.regeocode.formattedAddress;

                         cModel.myAddress('update',{data:value});

                         // server.my_address.update(item);
                       }else{

                       }
                   });
                 })


              });

          }else{

          }
          if(typeof(success)==="function"){
            success(res);
          }
        })
        .catch(error => {
          _this.isLoading = 0;
          console.log(error)
        })
    },

    /**
     * 下接刷新
     */
    onRefresh(done) {
      this.getList(1);
      done(); // call done
    },

    /**
     * 通过高德查找地址
     */
    searchMapAddress (){
      var _this = this;
      var keyword = this.keyword;

      // 檢查是存在本地城市信息
      var local_city =  typeof(this.$store.state.localCity) != "undefined" && typeof(this.$store.state.localCity.city) == 'string' ?  this.$store.state.localCity.city : "";
      //使用高德地圖自動無成插件
      AMap.plugin('AMap.Autocomplete',function(){//回调函数
          //实例化Autocomplete
          var autoOptions = {
              city: local_city, //城市，默认全国
          };
          var autocomplete = new AMap.Autocomplete(autoOptions);
          autocomplete.search(keyword, function(status, result){
            if(status == 'complete'){
              _this.smListDatas = [];
              // console.log(result.tips);
               result.tips.forEach(function(value,index,arr){
                if(value.location.lat && value.location.lng){
                  let itemValue =  {
                    addressid:0,
                    addressname:value.name,
                    address:value.address,
                    district:value.district,
                    latitude:value.location.lat,
                    longtitude:value.location.lng,
                  }
                  _this.smListDatas.push(itemValue);
                }
              })
              // console.log(_this.smListDatas);
              // console.log(result.tips);

            }
          });
      });

    },

    /**
     * [onSelectAddress 当选择地址时]
     * @param  {index}  index     [数据索引]
     * @param  {Boolean} isFromMap [是否来自地图]
     */
    onSelectAddress (index,isFromMap){
      var _this = this;
      var data = isFromMap ? this.smListDatas[index] : this.listDatas[index]

      var  to = this.to;
      let formData = this.$store.state.routeFormData;
      //如果是选择起点和终点
      if(to=="start"||to=="end"){
        formData[to] = data;
        // console.log(formData);
        this.$store.commit('setRouteFormData',formData);
        this.$router.back();
      }
      //如果是修改公司和家的地址。
      if(to=="home"||to=="work"){
        // console.log(data);
        //取得用户信息
        var userData = this.$store.state.userData;
        //要提交的数据
        var  postData = {
          latitude:data.latitude,
          longtitude:data.longtitude,
          aid:data.addressid,
          name:data.addressname,
          from: to
        }
        //提交修改个人信息（公司或家地址）
        this.$tokenAxios.post(config.urls.editProfileAdress,postData).then(res => {
          var resData = res.data.data
          if(res.data.code === 0) {
            var data_n = data;
            data_n.addressid = typeof(resData.createAddress.addressid)!='undefined' ? resData.createAddress.addressid : data.addressid;
            data_n.addressname = data.addressname;
            if(to=='home'){
              data_n.listorder = 0;
              data_n.address_type = 'Home';
              userData.home_address = data.addressname;
              userData.home_address_id = data.addressid;
            }else if(to == 'work'){
              data_n.listorder = 1;
              data_n.address_type = 'Work';
              userData.company_address = data.addressname;
              userData.company_address_id = data.addressid;
            }
            //把更新的数据写回到 $store
            this.$store.commit('setUserData',userData);
            //把旧的移出home或work位置
            cModel.myAddress('only',{
              data:{'address_type':data_n.address_type},
              success:function(result,server){
                if(result){
                  var data_o = result;
                  data_o.listorder = 3;
                  data_o.address_type = 'Often'
                  cModel.myAddress('update',{data:data_o});
                }
                // return false;
                //把新的写进home或work位置
                if(typeof(resData.createAddress.addressid)!='undefined'){
                   cModel.myAddress('add',{data:data_n});
                }else{
                   cModel.myAddress('update',{data:data_n});
                }
                // console.log(GB_VAR['user_info']);
                _this.$vux.toast.text("更改成功");

              }
            })
          }else{
            _this.$vux.toast.text("更改失败，请稍候再试");

          }
          // _this.$router.push({name:'user_profile'});
          this.$router.back();
        })
        .catch(error => {
          _this.$vux.toast.text("更改失败，请稍候再试");
          this.$router.back();
          console.log(error)
        })

      }
    },

    goCreateAddress (){
      this.$router.push({name:'carpool_address_create',params: {to:this.to,keyword:this.keyword}})
    }

  },
  created () {
    // this.init();


    // this.$nextTick(function () {
    //  this.$refs['j-herblist-scrollBox'].addEventListener('scroll', this.listScroll); //监听滚动加载更多
    // })
  },
  activated (){
    this.getList();

    if(!this.$store.state.localCity){
      this.getCity();
    }
    this.to = this.$route.params.to;
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

</style>
