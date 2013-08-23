                                          cartAndOrder
============

                                       购物车和下单页面js
                                       
  

 1 @fileoverview ERP下单购物车
 2 @module erp/dianxiao
 3 @constructor Electrical
 4 @param config{object}
 5 @return {object}
 

KISSY.add('erp/dianxiao',function(S) {
	var D = S.DOM,
		E = S.Event;
	var isDaily = location.host.indexOf('hitao.com') > -1 ? 'hitao.com' : 'daily.hitao.net';
	var API = {
		search:'http://erp.'+isDaily+'/home/acrossPermission.htm?action=/orderform/AddTelOrderAction&event_submit_do_query_product=anything',
		orderDefault:'http://erp.'+isDaily+'/home/acrossPermission.htm?action=/orderform/AddOrderAction&event_submit_do_get_consignee_address=anything&_input_charset=UTF-8',
		addAddress:'http://erp.'+isDaily+'/home/acrossPermission.htm?action=/orderform/AddOrderAction&event_submit_do_save_new_consignee_address=anything&_input_charset=UTF-8',
		logistics:'http://erp.'+isDaily+'/home/acrossPermission.htm?action=/orderform/AddOrderAction&event_submit_do_get_supported_logistics=anything&_input_charset=UTF-8',
		order: 'http://erp.'+isDaily+'/home/acrossPermission.htm?action=/orderform/AddTelOrderAction&event_submit_do_add_order=something'
		
	};
	function Electrical(config) {
		var self = this;
		
		if(self._inited) {
			self._init(config);
			self._inited = false;
		}
	}
	Electrical.prototype = {

		constructor:Electrical,
		_inited: true,

		_init: function(cfg) {
			var self = this;
			E.remove(cfg,'click');
			E.add(cfg,'click',function(e) {
				var target = e.target;
				switch(true) {
					// 单选
					case D.hasClass(target,'J_CheckBoxItem'):
						 self._doSelectItem(e);
					break;

					// 全选
					case D.hasClass(target,'J_SelectAll'):
						 self._doSelectAll(target);
					break;

					// 删除一项
					case D.hasClass(target,'J_DelItem'):
						 self._doDelItem(e,target);
					break;

					// 选择网销宝贝
					case D.hasClass(target, 'J_Select_Baby'):
						 self._selectBaby(e);
					break;

					// 去结算
					case D.hasClass(target,'J_Go'):
						 self._totalGo(e);
					break;
					
					//批量删除
					case D.hasClass(target,'J_DelSelect'):
						 self._doBatchDel(target);
					break;

					default:
						break;
				}
			});
		},

		_count: 0,

		_price: 0,

		_amount: 0,

		_obj: {},
		
		_numArr: [],

		_postArr: [],
		
		_selectBabyArr:[],
		
		/*
		 * 选择一项
		 */
		_doSelectItem: function(e){
			e.stopPropagation();
			var self = this,
				target = e.target;
			/* 页面所有的单选框的长度
			 * 当选择一项时候 计数器加1 当计数器大于或者等于 单选框长度时候
			 * 全选的选择框全选 否则的话 不全选
			 */
			var trLists = D.query(".dx-list"),
				selectAlls = D.query(".J_SelectAll"),
				trElem = D.parent(target,2),
				priceVal = D.attr(D.get(".J_Subtotal",trElem),'data-value'),
				dxPrice = D.attr(D.get('.dx-price',trElem),'data-value'),
				changePrice = D.attr(D.get('.J_TextAmount',trElem),'value'),
				jAmount = D.val(D.get('.J_Amout',trElem)),
				listsArr = [],
				listsLen;

			S.each(trLists,function(list,index){
				if(!D.hasClass(list,'display')){
					listsArr.push(list);
				}
			});
			listsLen = listsArr.length;
			
			if(target.checked){
				target.checked = true;
				D.addClass(trElem,'selected');
				self._count++;
				if(self._count >= listsLen){
					S.each(selectAlls,function(item,index){
						item.checked = true;
					});
				}
			
			self._amount +=jAmount * 1;
			
			
			self._price += (dxPrice * 1 * jAmount + changePrice * 1);
			self._obj = {
				count: self._count,
				price: self._price,
				amount: self._amount
			};
			if(self._count > 0){
				var tparent = D.parent(target,5);
				D.hasClass(tparent,'un-go') && D.removeClass(tparent,'un-go');
			}
			
			/* 
			 * 选中一项时 计算下几款宝贝 几件商品 商品总价相应的信息
			 */
			
			 self._calculate(self._obj);
			 
			 /*
			 * 增加一项时候 小计值 发生相对应的变化。
			 */
			 D.html(D.get(".J_Subtotal",trElem),(dxPrice * jAmount * 1 + changePrice * 1).toFixed(2));
			 D.attr(D.get(".J_Subtotal",trElem),{"data-value":dxPrice * jAmount * 1 + changePrice * 1});
			}else{
				target.checked = false;
				D.hasClass(trElem,'selected') && D.removeClass(trElem,'selected');
				self._count--;
				if(self._count < listsLen){
					S.each(selectAlls,function(item,index){
						item.checked = false;
					});
				}
				if(self._count <= 0){
					var tparent = D.parent(target,5);
					!D.hasClass(tparent,'tparent') && D.addClass(tparent,'un-go');
				}
				var price2 = dxPrice * 1 * jAmount;
				
				self._price -= (price2 * 1 + changePrice * 1);
				
				self._amount -=jAmount * 1;
				self._obj = {
					count: self._count,
					price: self._price,
					amount: self._amount
				};
				/* 
				 * 删除一项时 计算下几款宝贝 几件商品 商品总价相应的信息
			     */
			    self._calculate(self._obj);
				/*
				 * 反选时候 获取tr属性 codeNum 的值 
				 * 然后遍历数组 self._postArr中的 code, 看反选时 如果数组任一项与反选时 的 codeNum相等的话 那么删除掉当前项
				 */
				var trCode = D.attr(trElem,"codeNum");
				for(var j = 0, jlen = self._postArr.length; j < jlen; j+=1){
					if(self._postArr[j].code == trCode){
						self._removeItem(self._postArr[j],self._postArr);
						break;
					}
					
				}
			}
			
		},
		
		/*
		 * 选择所有
		 */
		_doSelectAll: function(target){
			var self = this;
			self._price = 0;
			self._amount = 0;
			self._obj = {};
			var selectAlls = D.query(".J_SelectAll"),
				checkItems = D.query(".J_CheckBoxItem"),
				itemLen = checkItems.length,
				elemParent = D.parent(target,3),
				trElems = D.query('.dx-list',elemParent),
				subTotals = D.query('.J_Subtotal'),
				subAmount = D.query('.J_Amout');
			
			if(target.checked){
				S.each(trElems,function(trElem,index){
					!D.hasClass(trElem,'selected') && D.addClass(trElem,'selected');
				});
				// 给确认结算留钩子
				var tparent = D.parent(target,3);
				D.hasClass(tparent,'un-go') && D.removeClass(tparent,'un-go');

				selectAllFun();
				
			}else{
				S.each(trElems,function(trElem,index){
					D.hasClass(trElem,'selected') && D.removeClass(trElem,'selected');
				});
				// 给确认结算留钩子
				var tparent = D.parent(target,3);
				!D.hasClass(tparent,'un-go') && D.addClass(tparent,'un-go');

				selectBacks();
				/* 反选 全选按钮时候 直接让数组为空 */
			}

			/* 全选
			 * 全选时 计数器的长度等于 tr的长度
			 * 全选 计算价格 几款宝贝 几件商品 存入对象里
			 */
			function selectAllFun(){
				S.each(selectAlls,function(item,index){
					item.checked = true;	
				});
				S.each(checkItems,function(item,index){
					item.checked = true;
					self._count = itemLen; 
				});

				// 遍历小计 计算所有的价格
				S.each(subTotals,function(item,index){
					var itemVal = D.attr(item,'data-value');
					self._price += itemVal * 1;
				});
				// 遍历订购数量 计算所有的数量
				S.each(subAmount,function(item,index){
					var jAmount = D.val(item);
					self._amount += jAmount * 1;
				});
				self._obj = {
					count: self._count,
					price: self._price,
					amount: self._amount
				};
				
				/* 
				 * 全选时 计算下几款宝贝 几件商品 商品总价相应的信息
			     */
			    self._calculate(self._obj);
				//console.log(self._postArr);
			}

			/* 反选
			 * 反选时 计数器的长度重设为初始值 0
			 */
			function selectBacks(){
				S.each(selectAlls,function(item,index){
					item.checked = false;	
				});
				S.each(checkItems,function(item2,index2){
					item2.checked = false;
					self._count = 0;
				});
				// 遍历小计 计算所有的价格
				S.each(subTotals,function(item,index){
					var itemVal = D.attr(item,'data-value');
					self._price = 0;
				});
				// 遍历订购数量 计算所有的数量
				S.each(subAmount,function(item,index){
					var jAmount = D.val(item);
					self._amount = 0;
				});
				self._obj = {
					count: self._count,
					price: self._price,
					amount: self._amount
				};
				
				/* 
				 * 反选时 计算下几款宝贝 几件商品 商品总价相应的信息
			     */
			    self._calculate(self._obj);

				/* 反选时候 直接让数组为空 */
				self._postArr = [];
			}
			
		},
		
		/*
		 * 删除一项
		 */
		_doDelItem: function(e,target){
			var self = this;
			e.halt();
			var trElem = D.parent(target,2),
				priceVal = D.attr(D.get(".J_Subtotal",trElem),'data-value'),
				jAmount = D.val(D.get('.J_Amout',trElem)),
				selectItems = D.query(".J_CheckBoxItem",trElem),
				selectAll = D.query(".J_SelectAll");

			if (confirm('确定要删除该宝贝吗？')) {
				S.Anim(trElem, {opacity: 0}, 0.5, S.Easing.easeOut,function(){
					
					D.remove(trElem);
					S.each(selectItems,function(item){
						if(item.checked){
							self._price -= priceVal * 1;
							self._price *= 1;
							self._amount -=jAmount * 1;
							self._count--;
							if(self._count <= 0){
								S.each(selectAll,function(item,index){
									item.checked = false;
								});
							}
							self._obj = {
								count: self._count,
								price: self._price,
								amount: self._amount
							};
							self._calculate(self._obj);
						}
					});
				}).run();
			}
		},
		
		/*
		 * 选择网销宝贝
		 * 弹窗 点击输入条件查询时候 把相应的数据渲染出来
		 */
		_selectBaby: function(e){
			var self = this;
			e.halt();
			D.get("#baby-window") && D.hasClass(D.get("#baby-window"),'hidden') && D.removeClass(D.get("#baby-window"),'hidden');
			self._showDialog('#baby-window');

			E.remove("#baby-window",'click');
			E.add('#baby-window','click',function(ev){
				var target = ev.target;
				
				if(D.hasClass(target,"J_Search")){
					self._searchData();

				}else if(D.hasClass(target,'add-btn')){
					self._addBaby(ev);

				}else if(D.hasClass(target,'closed-btn')){
					self._closedWindow('#baby-window');
					D.get('#page-content-st') && D.html(D.get("#page-content-st"),'');
					D.get('#page') && D.html(D.get("#page"),'');
				}
			});
		},
		/*
		 * 弹窗居中显示
		 */
		_showDialog: function(html){
			var viewWidth,viewHeight,
				scrollLeft,scrollTop,
				elemWidth,elemHeight;
			viewWidth = D.viewportWidth();
			viewHeight = D.viewportHeight();
			scrollLeft = D.scrollLeft(window);
			scrollTop = D.scrollTop(window);
			elemWidth = D.width(html);
			elemHeight = D.height(html);
			var left = (viewWidth - elemWidth)/2 + scrollLeft, 
				top = (viewHeight - elemHeight)/2 + scrollTop;
			D.css(html,{'position':'absolute',"left":left,"top":top,'z-index':100});
		},
		_closedWindow: function(elem){
			!D.hasClass(elem,'hidden') && D.addClass(elem,'hidden');
			
		},
		_searchData: function(){
			var self = this;
			self._selectBabyArr = [];
			self._selectBabyNames = [];
				
			/*
			 * 发jsonp请求 把相应的数据渲染出来
			 */
			   KISSY.use('gallery/pagination/1.1/pagination, menubutton', function (S, P, MenuButton) {
					var pagesize = 10,
						babyId = S.trim(D.val('#babyId')),
						itemName = S.trim(D.val('.J_Input')),
						itemCode = S.trim(D.val('#itemCode'));
					E.on('#babyId','valuechange',function(e){
					   var newVal = e.newVal;
					   if(!/^\d+$/.test(newVal)){
						   D.val('#babyId','');
					   }
					});
					var	pagination = new P({
							container: '#page',
							totalPage: 100,
							template: S.one('#default-pagination-tpl-2').html().replace(/@/g,"#"),
							callback: function (idx, pg, ready) {
								S.ajax({
									url: API.search,
									dataType: 'jsonp',
									data: {
										'per_page': pagesize,
										'format': 'json',
										'page': idx,
										'auctionId':babyId,
										'itemName': encodeURIComponent(itemName),
										'itemCode':itemCode
									},
									jsonp: 'jsoncallback',
									success: function (data2) {
										var html = '',
											rows = data2.rows;
									  for(var j = 0, jlen = rows.length; j < jlen; j+=1){
										  html+='<tr class="J_Baby_list" codeNum="'+rows[j].itemCode+'">' + 
													'<td>' +
														'<div class="checkbox">' + 
															'<input type="checkBox" class="selectBaby"/>'+
														'</div>' +
													 '</td>' +
													 '<td>' +
														 '<div class="J_BabyID">'+rows[j].auctionId+'</div>'+
													 '</td>' +
													 '<td>' +
														 '<div class="J_BabyCode" value="'+rows[j].itemCode+'" title="'+rows[j].itemCode+'">'+rows[j].itemCode+'</div>'+
													 '</td>' +
													 '<td>' +
														 '<div class="J_BabyName" value="'+rows[j].itemName+'" title="'+rows[j].itemName+'">'+rows[j].itemName+'</div>'+
													 '</td>' +
													 '<td>' +
														 '<div class="J_BabyPrice" value="'+rows[j].yuanSellPrice+'">'+rows[j].yuanSellPrice+'</div>'+
													 '</td>'+
													 '<td>'+
														 '<div class="J_BabyBrand">'+rows[j].brandName+'</div>'+
													 '</td>' +
													 '<td>' +
														'<div class="J_BabyPlace">'+rows[j].producer+'</div>'+
													 '</td>'+
													 '<td>' +
														 '<div class="J_BabyDate">'+rows[j].guaranteeDate+'</div>' +
													 '</td>'+
													 '<td>'+
														 '<div class="J_BabyStore">'+rows[j].onlineStock+'</div>'+
													 '</td>'+
												  '</tr>';
									  }
									  D.get('#page-content-st') && D.html(D.get('#page-content-st'),html,false,function(){
										/*
										 * 选择宝贝 选中一行时候 存储到数组里面去
										 * 构造数组 类似如下：
										 * [{itemCode:'',itemName:'',hitaoPrice:''},{},{}]
										 */
										 var selectBabys = D.query(".selectBaby");

										 // 去掉数组中的重复项
										 var unique = function(arr){
											var ret = [],
												hash = {},
												key;
											for(var i = 0,ilen = arr.length; i < ilen; i+=1) {
												var curItem = arr[i].itemCode,
													key = curItem;
												if(hash[key] !== 1){
													ret.push(arr[i]);
													hash[key] = 1;
												}
											}
											return ret;
										 };
										 S.each(selectBabys,function(item,index){
											  E.remove(item,'click');
											  E.add(item,'click',function(e){
												  var target = e.target,
													  trElem = D.parent(target,3),
													  itemCode = D.attr(D.get(".J_BabyCode",trElem),"value"),
													  itemName = D.attr(D.get('.J_BabyName',trElem),"value"),
													  hitaoPrice = D.attr(D.get('.J_BabyPrice',trElem),"value");
												  if(target.checked){
													  target.checked = true;
													  self._selectBabyArr.push({itemCode:itemCode,itemName:itemName,hitaoPrice:hitaoPrice});
													  self._selectBabyArr.join(',');
													  self._selectBabyArr = unique(self._selectBabyArr);
													 
												  }else{
													  target.checked = false;
													  var tparentVal = D.attr(D.parent(target,3),"codeNum");
													  for(var j = 0, jlen = self._selectBabyArr.length; j < jlen; j+=1){
														if(self._selectBabyArr[j].itemCode == tparentVal){
															self._removeItem(self._selectBabyArr[j],self._selectBabyArr);
															break;
														  }
													  }
												  }
											  });
										 });
										
									  });
										pagination.set("totalPage",data2.total);
										var tpage = pagination.get('totalPage');
										if(tpage == 1){
											D.get('.pg-next') && D.css('.pg-next',{'cursor':'text'});
											D.get('.pg-first') && D.css('.pg-first',{'cursor':'text'});
											D.get('.pg-last') && D.css('.pg-last',{'cursor':'text'});
											D.get('.J_jumpToBtn') && D.css('.J_jumpToBtn',{'cursor':'default'});
										}
										// 加载完内容后, 通知下分页器更新加载状态
										ready(idx);
									}
									
								});
							},
							events: {
							  'J_jumpToBtn': {
								  click: function (e) {
									  var pg = this;
									  pg.page(pg.get('container').one('#J_jumpTo').val());
								  }
							  }
							},
					}),
					newSelect = function () {
						var select = MenuButton.Select.decorate('#decorateSelect', {
							prefixCls: "c2c-",
							menuAlign: {
								offset: [0, -1]
							},
							width: 82,
							menuCfg: {
								height: 150,
								elStyle: {
									overflow: "auto",
									overflowX: "hidden"
								}
							}
						});
						return select;
					},
					select = newSelect();
					
					pagination.on('beforePageChange', function (v) {

						// 更新分页器前, 销毁其他组件
						select && select.destroy();
					});
					pagination.on('afterPageChange', function (v) {
						var tpage = pagination.get('totalPage');
						if(v.idx > tpage){
						   return;
						}
						
						// 重新初始化 select
						select = newSelect();
						
					});
					
			   });
		},
		_windowArr: [],
		_selectBabyList: [],
		
		/*
		 * 弹窗口的列表添加到页面上来
		 */
		_addBaby: function(e){
			e.halt();
			var self = this,
				temp = false,
				tempArr = [];
			/*
			 * 增加宝贝之前 先遍历下页面所有的tr 如果有与选择相同tr项的话 那么不增加
			 * 同时 在相应的订购数量加1 否则把相应的项依次加到后面去
			 */
			var pageLists = D.query("#J_Tbody .dx-list");
			if(pageLists.length === 0){
				addHTML();
			}else{
				for(var k = 0, klen = pageLists.length; k < klen; k+=1){
					var codeNum = D.attr(pageLists[k],'codeNum');
					tempArr[k] = codeNum;
				}
				
				self._windowArr = [];
				for(var i = 0, ilen = self._selectBabyArr.length; i < ilen; i+=1){
					var itemCode = self._selectBabyArr[i].itemCode;
					self._windowArr[i] = itemCode;
				}
				for(var n = 0, nlen = self._windowArr.length; n < nlen; n+=1){
					var index = S.indexOf(self._windowArr[n],tempArr);
					if(index > -1){
						getTR(index);
					}else{
						addHTML();
					}
				}
			}
			/*
			 * 再遍历下页面tr项 如果与传进来的编码相同的话 那么获取当前的项 当前的项输入框加1
			 */
			function getTR(index){
				var amount = D.attr(D.get('.J_Amout',pageLists[index]),'value');
				amount = amount*1 + 1;
				D.attr(D.get('.J_Amout',pageLists[index]),{'value':amount});
				D.val(D.get('.J_Amout',pageLists[index]),amount);

				// 加1的同时 小计跟着变化 商品总价跟着变化
				var hitaoPrice = D.attr(D.get('.dx-price',pageLists[index]),'data-value');
				D.html(D.get('.J_Subtotal',pageLists[index]),(hitaoPrice * amount * 1).toFixed(2));
				D.attr(D.get('.J_Subtotal',pageLists[index]),{"data-value":hitaoPrice * amount * 1});
				
				self._closedWindow("#baby-window");
				D.html("#page-content-st","");
				D.html("#page","");

				// 遍历 订购数量全部加起来(先让self._amount 为0) 价格全部加起来
				self._amount = 0;
				self._price = 0;
				var dxlists = D.query("#J_Tbody .dx-list"),
					selectAll = D.query(".J_SelectAll"),
					dxboxs = D.query('.J_CheckBoxItem');
				S.each(dxlists,function(dxlist,dxindex){
					var amount = D.get('.J_Amout',dxlist),
						aval = D.attr(amount,"value"),
						subtotal = D.attr(D.get('.J_Subtotal',dxlist),'data-value');
					self._price += subtotal * 1;	
					self._amount += aval * 1;
				});
				//页面已增加一项 页面默认项全选
				S.each(dxboxs,function(item,index){
					item.checked = true;
				});
				S.each(selectAll,function(item,index){
					item.checked = true;
				});

				self._obj = {
					count: self._count,
					price: self._price,
					amount: self._amount
				};
				/* 
				 * 计算下几款宝贝 几件商品 商品总价相应的信息
				 */
				self._calculate(self._obj);
			}
			function addHTML(){
				for(var a = 0, alen = pageLists.length; a < alen; a+=1){
					var codeNum = D.attr(pageLists[a],'codeNum');
					for(var j = 0, jlen = self._selectBabyArr.length; j < jlen; j+=1){
						if(self._selectBabyArr[j].itemCode == codeNum){
							self._removeItem(self._selectBabyArr[j],self._selectBabyArr);
							break;
						}
					}
				}
				var html = "";
				for(var k = 0, klen = self._selectBabyArr.length; k < klen; k++){
					html += '<tr class="dx-list" attr="0" codeNum = "'+self._selectBabyArr[k].itemCode+'">'+
								'<td class="dx-check padding8">' + 
									'<input type="checkbox" name="cartIds" class="J_CheckBoxItem">'+
								'</td>' +
								'<td class="dx-code padding8" title="'+self._selectBabyArr[k].itemCode+'">'+self._selectBabyArr[k].itemCode+'</td>'+
								'<td class="dx-name padding8">' +
									'<div class="dx-desc" title="'+self._selectBabyArr[k].itemName+'">'+self._selectBabyArr[k].itemName+'</div>'+
								'</td>'+
								'<td class="dx-price padding8" data-value="'+self._selectBabyArr[k].hitaoPrice+'">'+self._selectBabyArr[k].hitaoPrice+'</td>'+
								'<td class="dx-amount padding8">' +
									'<input type="text" class="text-amount J_Amout" value="1">'+
								'</td>'+
								'<td class="dx-rprice padding8">'+
									'<input type="text" class="text-amount J_TextAmount" value="0.00">'+
								'</td>'+
								'<td class="dx-total-price padding8">'+
									'<span class="J_Subtotal" data-value="'+self._selectBabyArr[k].hitaoPrice+'">'+self._selectBabyArr[k].hitaoPrice+'</span>'+
									'<a href="#" class="J_DelItem">删除</a>'+
								'</td>'+
							'</tr>';
					
				}
				D.append(D.create(html),D.get("#J_Tbody"));
				var dxboxs = D.query('.J_CheckBoxItem'),
					selectAll = D.query(".J_SelectAll"),
					dxlists = D.query('.dx-list'),
					dxLen = dxlists.length;

				S.each(dxboxs,function(item,index){
					item.checked = true;
				});
				S.each(dxlists,function(list,index){
					!D.hasClass(list,'selected') && D.addClass(list,'selected');
				});
				S.each(selectAll,function(item,index){
					item.checked = true;
				});
				// 给确认结算留钩子
				var tparent = D.get('#J_CartEnable');
				D.hasClass(tparent,'un-go') && D.removeClass(tparent,'un-go');

				self._closedWindow('#baby-window');
				D.html("#page-content-st","");
				D.html("#page","");
				self._selectBabyArr = [];
				self._count = dxLen;
				
				// 遍历 订购数量全部加起来(先让self._amount 为0) 价格全部加起来
				self._amount = 0;
				self._price = 0;
				S.each(dxlists,function(dxlist,dxindex){
					var amount = D.get('.J_Amout',dxlist),
						aval = D.attr(amount,"value"),
						subtotal = D.attr(D.get('.J_Subtotal',dxlist),'data-value');
					self._price += subtotal * 1;	
					self._amount += aval * 1;
				});

				self._obj = {
					count: self._count,
					price: self._price,
					amount: self._amount
				};
				self._calculate(self._obj);
			}
			/*
			 * 订购数量改变时候 触发事件
			 * 根据valuechange 事件来触发
			 */
			var buyAmounts = D.query(".J_Amout");
			S.each(buyAmounts,function(amountItem,index){
				E.remove(amountItem,'valuechange');
				E.add(amountItem,'valuechange',function(e){
					var target = e.target,
						trElem = D.parent(target,2),
						thisOldNum = e.prevVal,
						thisNum = e.newVal,
						hitaoPrice = D.attr(D.get(".dx-price",trElem),'data-value'),
						changePrice = D.attr(D.get(".J_TextAmount",trElem),'value'),
						subTotalElem = D.get(".J_Subtotal",trElem),
						tempAmount = 0;

					if(/^\d+$/.test(thisNum)){
						//当前的属性value 要改下
						D.attr(target,{"value":thisNum});
						//最终价格
						var endPrice = (hitaoPrice * thisNum * 1 + changePrice * 1).toFixed(2);
						D.html(subTotalElem,endPrice);
						//小计属性value更新下
						D.attr(subTotalElem,{"data-value":endPrice});
						
						//判断tr是否选中 是的话 计算下 否则 不计算
						if(D.hasClass(trElem,'selected')){
							
							if(!/^\d+$/.test(thisOldNum)){
								thisOldNum = 0;
								thisNum = 0;
							}
							
							tempAmount +=(thisNum * 1) - (thisOldNum *1);
							self._amount += tempAmount;
							self._price += hitaoPrice * (thisNum * 1 - thisOldNum *1) * 1;
							self._obj = {
								count: self._count,
								price: self._price,
								amount: self._amount
							};
							/* 
							 * 订购数量改变时候 计算下几款宝贝 几件商品 商品总价相应的信息
							 */
							self._calculate(self._obj);
						}
					}else{
						alert("订购数量应为数字或者不能为负数！否则不生效");
						D.val(target,thisOldNum);
						return;
					}
				});
			});
			/*
			 * 调价值改变的时候 触发事件
			 * 根据valuechange 事件来触发
			 */
			 var textChangeValues = D.query(".J_TextAmount");
			 S.each(textChangeValues,function(textValItem,index){
				E.remove(textValItem,'valuechange');
				E.add(textValItem,'valuechange',function(e){
					var target = e.target,
						trElem = D.parent(target,2),
						hitaoPrice = D.attr(D.get(".dx-price",trElem),'data-value'),
						thisOldNum = e.prevVal,
						thisNum = e.newVal,
						orderNum = D.attr(D.get('.J_Amout',trElem),"value"),
						subTotalElem = D.get(".J_Subtotal",trElem);
					
					
					
					//当前的属性value 要改下
					D.attr(target,{"value":thisNum});

					if(!/^[+-]?\d+(?:\.\d+)?$/.test(thisOldNum)){	
						thisOldNum = 0;	
					}

					if(!/^[+-]?\d+(?:\.\d+)?$/.test(thisNum)){
						thisNum = 0;	
					}
					

					//最终价格
					var endPrice = (hitaoPrice * orderNum * 1 + thisNum * 1).toFixed(2),
						tempEndPrice = (hitaoPrice * orderNum * 1).toFixed(2);
					
					if(endPrice < 0){
						alert('调价不能大于原价！请修改相对应的值');
						D.val(target,0);
						D.attr(target,{"value":0});
						D.html(subTotalElem,tempEndPrice);
						D.attr(subTotalElem,{"data-value":tempEndPrice});
						//判断tr是否选中 是的话 计算下 否则 不计算
						if(D.hasClass(trElem,'selected')){
							self._price += Math.abs(thisOldNum * 1);
							self._obj = {
								count: self._count,
								price: self._price,
								amount: self._amount
							};
							/* 
							 * 订购数量改变时候 计算下几款宝贝 几件商品 商品总价相应的信息
							 */
							self._calculate(self._obj);
						}
					}else{
						D.html(subTotalElem,endPrice);
						//小计属性value更新下
						D.attr(subTotalElem,{"data-value":endPrice});
						//判断tr是否选中 是的话 计算下 否则 不计算
						if(D.hasClass(trElem,'selected')){
							self._price += (thisNum * 1 - thisOldNum * 1);
							self._obj = {
								count: self._count,
								price: self._price,
								amount: self._amount
							};
							/* 
							 * 订购数量改变时候 计算下几款宝贝 几件商品 商品总价相应的信息
							 */
							self._calculate(self._obj);
						}
					}	
					
				});
			 });
		},
		_orderLists: [],
		_orderMessages:[],
		_totalGo: function(e){
			var self = this,
				target = e.target;
			e.halt();
			self._orderLists = [];
			/*
			 * 当点击确认结算时候 遍历页面所有已选择的tr项 把相应的信息已数组的形式保存 如下
			 * 参数以数组的形式传过去
			 * 构造对象如下 {code:'',title:'',hitaoPrice:'',orderNum:'',changePrice:'',totalPrice:''}
			 */
			var targetParent = D.parent(target,2),
				dxlists = D.query(".dx-list",targetParent);
			if(dxlists.length === 0 || D.hasClass(targetParent,'un-go')){
				return;
			}
			S.each(dxlists,function(item,index){
				if(D.hasClass(item,'selected')){
					var code = S.trim(D.html(D.get('.dx-code',item))),
						title = S.trim(D.html(D.get('.dx-desc',item))),
						hitaoPrice = D.attr(D.get('.dx-price',item),'data-value'),
						orderNum = D.attr(D.get('.J_Amout',item),'value'),
						changePrice = D.attr(D.get('.J_TextAmount',item),'value'),
						totalPrice = D.attr(D.get('.J_Subtotal',item),'data-value'),
						totalAmount = hitaoPrice * orderNum;
					self._orderLists.push({code:code,title:title,hitaoPrice:hitaoPrice,orderNum:orderNum,changePrice:changePrice,totalPrice:totalPrice});
					self._orderLists.join(',');
					//self._orderMessages = [];
					self._orderMessages.push({itemCode:code,itemName:title,sellPrice:hitaoPrice,orderCount:orderNum,adjustFee:changePrice,itemAmout:totalPrice,totalAmount:totalAmount});
				}
			});
			
			/*
			 * 当前的页面隐藏 下单页面显示 
			 */
			 D.get('.J_Tab_hidden') && !D.hasClass('.J_Tab_hidden','hidden') && D.addClass('.J_Tab_hidden','hidden');
			 D.get('.container-order') && D.hasClass('.container-order','hidden') && D.removeClass('.container-order','hidden');
			 // 调用订单js
			 self._orderInit();
		},
		
		/*
		 * 批量删除
		 * 如果页面上未选择任何一项时候 弹出确认框 请选择宝贝
		 * 否则的话 弹出确认框 确定要删除选中的宝贝吗？等字样
		 */
		_doBatchDel: function(target){
			var self = this,
				selectAll = D.query(".J_SelectAll"),
				dxLists = D.query("#J_CartEnable .dx-list");
			/* 
			 * 根据self._count的个数 来判断页面中是否选择
			 * 如果个数等于0的话 那么提示 请选择宝贝字样 返回
			 */
			if(self._count === 0){
				confirm('请选择宝贝');
				return;
			}
			S.each(selectAll,function(item,index){
				item.checked = false;
			});
			// 给确认结算留钩子
			var tparent = D.parent(target,2);
			!D.hasClass(tparent,'un-go') && D.addClass(tparent,'un-go');
			/* 
			 * 遍历tr
			 * 页面根据tr的属性attr 是否等于1 如是 说明已选择了项
			 * 否则的话 未选择
			 */
			S.each(dxLists,function(item,index){
				var jAmount = D.val(D.get('.J_Amout',item));
					
				if(D.hasClass(item,'selected')){
					S.Anim(item, {opacity: 0}, 0.5, S.Easing.easeOut,function(){
						D.remove(item);
						self._numArr.push(item);
						self._selectAllState(dxLists);
						
						self._price = 0; // 批量删除后 价格直接为0;
						self._amount -=jAmount * 1;
						self._count--;
						self._obj = {
							count: self._count,
							price: self._price,
							amount: self._amount
						};
						self._calculate(self._obj);
						
					}).run();
				}
			});
		},
		
		/*
		 * 当页面选择最后一项tr也被删掉的时候 全选复选框不勾选
		 */
		_selectAllState: function(dxLists){
			var self = this,
				selectAll = D.query(".J_SelectAll"),
				selectAllLen = selectAll.length;
			if(self._numArr.length >= selectAllLen){
				S.each(selectAll,function(item,index){
					item.checked = false;
				});
			}
			
		},
		/**
         * 重新计算金额,几款宝贝,几件商品
         */
		_calculate: function(obj){
			
			
			// 几款宝贝
			D.get(".J_Num") && S.trim(D.html(D.get(".J_Num"),obj.count));
			
			
			// 几件商品
			D.get(".J_Packages") && S.trim(D.html(D.get(".J_Packages"),obj.amount));
			
			//商品总价
			D.get("#J_Total") && S.trim(D.html(D.get("#J_Total"),obj.price.toFixed(2)));
			
		},
		
		/* 删除数组里面的某一项 */
		_removeItem: function(item,arr){
			var index = S.indexOf(item,arr);
			if(index > -1){
				arr.splice(index,1);
			}
		},
		//先写死 测试
		userId: encodeURIComponent(D.attr('#user','data-value')),
		
		// 订单后面后 是否是点击返回购物车后 的一个标志
		_callCartFlag: true,
		/*
		 * 订单页面 
		 * 发jsonp请求 默认渲染收获地址 及 配送信息 及确认商品订单
		 * 配送信息请求带第一次请求返回的 id 过去
		 */
		_orderInit: function(){
			var self = this;
			var addressList = D.get('#address-list'),
				orderSelect = D.get("#order-select"), //配送公司下拉
				orderHTML = "";
			if(self._callCartFlag){
				S.ajax({
					url:API.orderDefault,
					dataType: 'jsonp',
					jsonp: 'callback',
					data:{
						userId: self.userId
					},
					success: function(data){
						if(data.list.length > 0){
							var dataLists = data.list,
								invoice = data.invoice, // 收货人
								arriveable = data.arriveable, //物流公司是否为空 0 表示为空 1 表示不为空
								dataLogistics = data.logistics,
								html = "",
								renderHTML = "";
							  for(var i = 0, ilen = dataLists.length; i < ilen; i+=1){
								  html += '<li class="addr-list">' +
											   '<input type="radio" value="'+dataLists[i].id+'" class="J_Change_Addr" name="addressId"/>' +
											   '<label for="label" class="label">'+ dataLists[i].address + 
													'<em class="tphone">'+dataLists[i].phone+'</em>' +
											   '</label>'
											'</li>';
							  }
							  D.html(addressList,'');
							  D.append(D.create(html),addressList);
							  
							  // 动态添加收货人
							  D.get('#tickectTop') && D.val('#tickectTop',invoice);
							
							  // 默认第一个选中
							  var firstList = D.get("#address-list .addr-list"),
								  firstInput = D.get('.J_Change_Addr',firstList);
							  firstInput.checked = true;
							  

							  if(dataLogistics.length > 0){
								  // 默认情况下把底部的配送公司渲染出来
								  for(var j = 0, jlen = dataLogistics.length; j < jlen; j+=1){
									  renderHTML += '<option value="'+dataLogistics[j].logisticsId+'" data-value="'+dataLogistics[j].logisticsName+'">'+dataLogistics[j].logisticsName+'</option>';  
								  }
								  D.html(orderSelect,renderHTML,false,function(){
									  var firArea = D.get('option',orderSelect);
									  firArea.selected = true; // 设置默认第一项 不设置的话 选择最后一项去了
								  });
								  D.attr('#submitOrder',{"disabled":false});
								  D.hasClass('#submitOrder','isclick') && D.removeClass('#submitOrder','isclick');
							  }else{
								 if(arriveable == "0"){
									 
									 D.hasClass('.wuliu-tips','hidden') && D.removeClass('.wuliu-tips','hidden');
									 D.attr('#submitOrder',{"disabled":true});
									!D.hasClass('#submitOrder','isclick') && D.addClass('#submitOrder','isclick');
								 }
							  }
						} 	
					}
				});
				 // 切换订单信息要调用的函数
				 self._changeAddr();
			}
			
			
			//默认情况下 渲染确认商品订单信息
			for(var k = 0, klen = self._orderLists.length; k < klen; k+=1){
				orderHTML += '<tr codenum="'+self._orderLists[k].code+'" attr="0" class="order-list">'+
								'<td class="order-code padding8">'+self._orderLists[k].code+'</td>'+
								'<td class="order-name padding8">'+
									'<div class="order-desc">'+self._orderLists[k].title+'</div>'+
								'</td>'+
								'<td data-value="'+self._orderLists[k].hitaoPrice+'" class="order-price padding8">'+self._orderLists[k].hitaoPrice+'</td>'+
								'<td class="order-amount padding8">'+self._orderLists[k].orderNum+'</td>'+
								'<td class="order-rprice padding8">'+self._orderLists[k].changePrice+'</td>'+
								'<td class="order-total-price padding8">'+
									'<span data-value="'+self._orderLists[k].totalPrice+'" class="J_Order_Subtotal">'+self._orderLists[k].totalPrice+'</span>'+
								'</td>'+
							'</tr>';
			}
			D.html("#J_Order_List",orderHTML,false,function(){
				var orderSubTotal = D.query('.J_Order_Subtotal'),
					totalPrice = 0,
					yunfei = D.val(".J_Order_Input");
				S.each(orderSubTotal,function(item,index){
					var price = S.trim(D.attr(item,'data-value'));
					totalPrice += price * 1;
				});
				totalPrice += yunfei * 1;
				D.get("#J_All_Price") && D.html('#J_All_Price',totalPrice.toFixed(2));
			});
			
			

			// 当输入框值发生改变的时候 重新计算下金额
			
			E.add('.J_Order_Input','valuechange',function(e){
				var value = e.newVal;
				if(/^[0-9]+(\.^[0-9]+$)?/.test(value)){
					D.get('.not-number') && !D.hasClass('.not-number','hidden') && D.addClass('.not-number','hidden');
					totalPriceFunc(value);
				}else{
					D.get('.not-number') && D.hasClass('.not-number','hidden') && D.removeClass('.not-number','hidden');
					return;
				}
			});
			function totalPriceFunc(value){
				var yTotalPrice = 0,
					orderSubTotal = D.query('.J_Order_Subtotal');
				S.each(orderSubTotal,function(item,index){
					var price = S.trim(D.attr(item,'data-value'));
					yTotalPrice += price * 1;
				});

				var price = yTotalPrice + value * 1; 
				D.get("#J_All_Price") && D.html('#J_All_Price',price.toFixed(2));
			}
			// 点击返回购物车 返回到购物车页面
			E.remove('.callback-go','click');
			E.add('.callback-go','click',function(e){
				e.halt();
				!D.hasClass('#order','hidden') && D.addClass('#order','hidden');
				D.hasClass('.J_Tab_hidden','hidden') && D.removeClass('.J_Tab_hidden','hidden');

				//返回购物车时候 让数组为空
				self._orderMessages = [];
				
				// 从订单页面返回购物车后 给个标志 为false时候 不需要重新渲染收获地址 订单信息
				self._callCartFlag = false;
			});
			// 点击确认订单时候 发post请求 返回成功或者失败页面
			E.remove('#submitOrder','click');
			E.add('#submitOrder','click',function(e){
				e.halt();
				var target = e.target;
				
				if(D.hasClass(target,'isclick')){
					return;
				}
				!D.hasClass(target,'isclick') && D.addClass(target,'isclick');
				
				D.attr(target,{"disabled":"disabled"});
				submitOrderFunc();
			});
			/*
			 * 提交订单时 获取所有的参数以post方式发送过去
			 * param addressId -> 地址,logisticsName -> 物流公司 orderArr 订单信息数组
			 */
			function submitOrderFunc(){
				if(!D.hasClass('#J_NewAddress','hidden')){
					alert('使用其他地址 请确定!!');
					return;
				}
				//提交订单时候 判断配送公司是否有无 无的话 禁止提交
				if(!D.hasClass('.wuliu-tips','hidden')){
					return;
				}
				var addrForm = S.IO.serialize("#addrForm"),
					totalHtml = S.trim(D.html('#J_All_Price')),
					totalPrice = totalHtml * 1;
					yuefei = S.trim(D.val('.J_Order_Input')) * 1,
					totalAllPrice = 0;
				// 遍历数组 取总价
				for(var t = 0, tlen = self._orderMessages.length; t < tlen; t+=1){
					var curValue = self._orderMessages[t].totalAmount;
					totalAllPrice += curValue * 1;
				}
				// 遍历下确认信息单选框选中的值
				var singleBoxs = D.query('.J_Change_Addr'),
					singleVal;
				for(var s = 0, slen = singleBoxs.length; s < slen; s+=1){
					if(singleBoxs[s].checked == true){
						singleVal = singleBoxs[s].value;
					}
				}
				// 遍历下发票信息单选框选中的值
				var singleTickets = D.query('.p-ticket'),
					singleOptions = D.query('#order-select option'),
					ticketVal = "",
					tickectName = "",
					orderOptions,
					logisticsName;
				for(var t = 0, tlen = singleTickets.length; t < tlen; t+=1){
					if(singleTickets[t].checked == true){
						ticketVal = singleTickets[t].value;
					}
				}
				
				tickectName = D.val('#tickectTop');
				// 遍历下配送物流公司选中的值
				for(var o = 0, olen = singleOptions.length; o < olen; o+=1){
					if(singleOptions[o].selected == true){
						orderOptions = singleOptions[o].value;
						logisticsName = D.attr(singleOptions[o],'data-value');
					}
				}
				/*
				 * post请求参数 一些隐藏域的值也要一并传过去
				 */
				var projectID = D.attr('#projectID','value'),
					userId = D.attr('#userId','value'),
					nick = D.attr('#nick','value'),
					projectName = D.attr('#projectName','value'),
					realName = D.attr('#realName','value');
				
				if(/^[0-9]*$/.test(yuefei)){
					S.ajax({
						url:API.order,
						dataType: 'jsonp',
						jsonp: 'jsoncallback',
						data: {
							'addressId': singleVal,
							'ticketName': encodeURIComponent(tickectName),
							'ticketVal': ticketVal,
							'logisticsId': orderOptions,
							'logisticsName': encodeURIComponent(logisticsName),
							'orderArrs':S.JSON.stringify(self._orderMessages),
							'actualAmount': totalPrice,
							'postFee': yuefei,
							'totalAmount': totalAllPrice,
							'projectID': projectID,
							'userId': userId,
							'nick': encodeURIComponent(nick),
							'projectName': encodeURIComponent(projectName),
							'realName': encodeURIComponent(realName)
						},
						type: 'post',
						success: function(data){
							if(data.succeed){
								!D.hasClass('.J_Tab_hidden','hidden') && D.addClass('.J_Tab_hidden','hidden');
								!D.hasClass('.container-order','hidden') && D.addClass('.container-order','hidden');
								D.hasClass('.success-message','hidden') && D.removeClass('.success-message','hidden');
								D.html('.success-message',data.message);
							}else{
								!D.hasClass('.J_Tab_hidden','hidden') && D.addClass('.J_Tab_hidden','hidden');
								!D.hasClass('.container-order','hidden') && D.addClass('.container-order','hidden');
								D.hasClass('.success-message','hidden') && D.removeClass('.success-message','hidden');
								D.html('.success-message',data.message);
							}	
						}
					});
				}else{
					return;
				}
				
				
			}

			/*
			 * 点击需不需要发票
			 */
			S.each('.p-ticket',function(item,index){
				E.remove(item,'click');
				E.add(item,'click',function(e){
					var target = e.target;
					if(D.hasClass(target,'must-ticket')){
						D.hasClass('.tickectTop','hidden') && D.removeClass('.tickectTop','hidden');
					}else if(D.hasClass(target,'no-must-ticket')){
						!D.hasClass('.tickectTop','hidden') && D.addClass('.tickectTop','hidden');
					}
				});
			});
			// 页面渲染时候 焦点在 不需要发票上 本来是写在页面上的 但是现在不方面改后台代码 所以直接动态设置
			D.attr('.no-must-ticket',{'checked':'true'});
			
		},
		// 省市区联动列表
		_tdist: window['tdist'] ? window['tdist'] : {},
		/*
		 * 处理切换订单信息
		 */
		_changeAddr: function(){
			var self = this;
			
			var changeAddrs = D.query('.J_Change_Addr'),
				newAddrs = S.all("#J_NewAddress");
				E.undelegate(document,'click','.J_Change_Addr');
				E.delegate(document,'click','.J_Change_Addr',function(e){
					var target = e.target;
					if(!D.hasClass(target,'J_OtherAddr')){
						var id = D.attr(target,'value');
						userSameAddr(newAddrs,id);
					}else{
						userOtherAddr(newAddrs);
					}
				});
			/* 
			 * 切换地址时 再次发请求 渲染下配送公司数据
			 * param {id}
			 */
			function userSameAddr(newAddrs,id){
				var orderSelect = D.get("#order-select"), //配送公司下拉
					renderHTML = "";
				if(confirm('更换地址后，您需要重新确认订单信息')){
					// 使用其他地址收起来
					newAddrs.slideUp();
					!D.hasClass('#J_NewAddress','hidden') && D.addClass('#J_NewAddress','hidden');
					// 初始化时候 先让下拉框为空
					D.html(orderSelect,'');

					S.jsonp(API.logistics + "&id="+id + "&timestamp="+S.now(),function(data){
						   var logs = data.logistics,
							   invoice = data.invoice, // 收货人
							   arriveable = data.arriveable; //是否有无配送公司 如无的话 提示显示
						   if(logs.length > 0){
							   for(var i = 0, ilen = logs.length; i < ilen; i+=1){
							       renderHTML += '<option value="'+logs[i].logisticsId+'" data-value="'+logs[i].logisticsName+'">'+logs[i].logisticsName+'</option>';
							   }
							   D.html(orderSelect,renderHTML,false,function(){
								   var firArea = D.get('option',orderSelect);
								   firArea.selected = true; // 设置默认第一项 不设置的话 选择最后一项去了
							   });
						   }
						  if(arriveable == "0"){
							  D.hasClass('.wuliu-tips','hidden') && D.removeClass('.wuliu-tips','hidden');
							  D.attr('#submitOrder',{"disabled":true});
							  !D.hasClass('#submitOrder','isclick') && D.addClass('#submitOrder','isclick');
						  }else{
							  !D.hasClass('.wuliu-tips','hidden') && D.addClass('.wuliu-tips','hidden');
							  D.attr('#submitOrder',{"disabled":false});
							  D.hasClass('#submitOrder','isclick') && D.removeClass('#submitOrder','isclick');
						  }
						  D.val('#tickectTop',invoice);
						      
					});
					return true;
				}else{
					return false;
				}
			}
			/* 使用其他地址时候 展开详细信息*/
			function userOtherAddr(newAddrs){
				newAddrs.slideDown();
				D.hasClass('#J_NewAddress','hidden') && D.removeClass('#J_NewAddress','hidden');
				// 地址联动
				newAddrInit(newAddrs);
				
				//验证表单
				valiateForm();
			}

			// 地址联动
			function newAddrInit(newAddrs){
				E.remove(newAddrs,'click');
				E.add(newAddrs,'click',function(e){
					var target = e.target;
					if(D.hasClass(target,'address-prov')){
						provFunc(target);
					}
				});
			}
			// 省份值改变时候 渲染所有的市
			function provFunc(target){
				E.remove(target,'change');
				E.add(target,'change',function(e){
					var tagVal = e.target.value,
						cityEl = D.get("#n_city_select"),
						html = "";

					if(tagVal){
						// 删除类名 可以根据这个类名判断是否选择了省份 如没有 说明已选择了
						D.hasClass(target,'disabled') && D.removeClass(target,'disabled');
					}else{
						!D.hasClass(target,'disabled') && D.addClass(target,'disabled');
					}
					
					var arrLists = self._linkage(tagVal);
						S.each(arrLists, function(item,index){
							html += '<option value="'+item.val+'">'+item.name[0]+'</option>';
						});
						cityEl && D.html(cityEl,html,false,function(){
							var firOption = D.get('option',cityEl),
								firVal = D.attr(firOption,'value'),
								areas = D.get('#n_area_select'),
								cityHTML = "";
							if(firOption){
								firOption.selected = true; // 设置默认第一项 不设置的话 选择最后一项去了
							}
							

							// 同时渲染第一项市相对应的区
							var cityLists = self._linkage(firVal);
							S.each(cityLists,function(area,index){
								cityHTML += '<option value="'+area.val+'">'+area.name[0]+'</option>';
							});
							areas && D.html(areas,cityHTML,false,function(){
								var firArea = D.get('option',areas);
								if(firArea){
									firArea.selected = true; // 设置默认第一项 不设置的话 选择最后一项去了
								}
								
							});

							// 当市改变时候
							cityFunc(cityEl,areas);
						});
					
					
				});
			}
			// 市值改变的时候 渲染所有的区
			function cityFunc(cityEl,areas){
				E.remove(cityEl,'change');
				E.add(cityEl,'change',function(e){
					var tagVal = e.target.value,
						areaLists = self._linkage(tagVal),
						areaHTML = "";
					S.each(areaLists,function(area,index){
						areaHTML += '<option value="'+area.val+'">'+area.name[0]+'</option>';
					});
					areas && D.html(areas,areaHTML,false,function(){
						var firArea = D.get('option',areas);
						firArea.selected = true; // 设置默认第一项 不设置的话 选择最后一项去了
					});
				});
			}

			//表单验证
			function valiateForm(){
				var provEl = D.get('#n_prov_select'), //省份
					cityEl = D.get('#n_city_select'), // 市
					areaEl = D.get("#n_area_select"), //区域
					addSureEl = D.get('#J_AddSure'),
					postCodeEl = D.get("#J_postCode"), // 邮政编码
					streetEl = D.get("#deliverAddress"), // 街道地址
					nameEl = D.get("#deliverName"), // 收货人姓名
					phoneSectionEl = D.get("#phoneSection"), // 区号
					phoneCodeEl = D.get("#phoneCode"), // 电话号码
					phoneExtEl = D.get("#phoneExt"), // 分机
					mobileEl = D.get("#deliverPhoneBak"); //手机号码

				E.remove(addSureEl,'click');
				E.add(addSureEl,'click',function(e){
					e.halt();
					var postCodeVal = S.trim(D.val(postCodeEl)),
						parentPost = D.parent(postCodeEl,2),
						errCode = D.get('.err-code',parentPost),

						streetVal = S.trim(D.val(streetEl)),
						parentStree = D.parent(streetEl,2),
						errStree = D.get('.err-code',parentStree),
						
						nameVal = S.trim(D.val(nameEl)),
						parentName = D.parent(nameEl,2),
						errName = D.get('.err-code',parentName);
					/* 下面一系列验证 */
					if(D.hasClass(provEl,'disabled')){
						var parentEl = D.parent(provEl,2),
							errCode2 = D.get('.err-code',parentEl);
						D.html(errCode2,'请选择省份！');
						return false;
					}
					
					if(!/^.{5,60}$/.test(streetVal) || /^\d+$/.test(streetVal)){
						D.html(errStree,'请填写街道地址,至少5个字,最多不能超过60个字，不能全部为数字或字母');
						return false;
					}else{
						D.html(errStree,'');
					}
					
					if(!/^.{2,15}$/.test(nameVal) || /null/.test(nameVal)){
						D.html(errName,'请填写正确姓名,至少2个字，最多15个字且其中不能有null字样');
						return false;
					}else{
						D.html(errName,'');
					}
					if(S.trim(D.val(phoneSectionEl)) + S.trim(D.val(phoneCodeEl)) + S.trim(D.val(phoneExtEl)) + S.trim(D.val(mobileEl)) === ''){
						D.html(".J_Err_Phone",'电话和手机至少填写一个');
						return false;
					}else{
						if(S.trim(D.val(phoneSectionEl)) + S.trim(D.val(phoneCodeEl)) + S.trim(D.val(phoneExtEl))){
							if(!/^\d{3,6}$/.test(S.trim(D.val(phoneSectionEl)))){
								D.html(".J_Err_Phone",'区号必须是3-6位数字');
								return false;
							}
							if(S.trim(D.val(phoneExtEl)) && !/^\d{1,6}$/.test(S.trim(D.val(phoneExtEl)))){
								D.html(".J_Err_Phone",'分机号必须是1-6位数字');
								return false;
							}
							if(!/^\d{5,10}$/.test(S.trim(D.val(phoneCodeEl)))){
								D.html(".J_Err_Phone",'主机号码必须是5-10位数字');
							}
						}
						D.html(".J_Err_Phone",'');
					}
					var provSelectVal = provEl.options[provEl.selectedIndex].text,
						citySelectVal = cityEl.options[cityEl.selectedIndex].text,
						areaSelectVal = areaEl.options[areaEl.selectedIndex].text,
						postCodeVal = S.trim(D.val(postCodeEl)),
						streetVal = S.trim(D.val(streetEl)),
						nameVal = S.trim(D.val(nameEl)),
						telphoneVal = S.trim(D.val(phoneSectionEl)) + '-' + S.trim(D.val(phoneCodeEl)) + '-' + S.trim(D.val(phoneExtEl));
						mobileVal = S.trim(D.val(mobileEl));

					/** 验证完后发请求 **/
					S.ajax({
						url:API.addAddress,
						dataType:'jsonp',
						data: {
							"userId": self.userId,
							"province": encodeURIComponent(provSelectVal),
							"city": encodeURIComponent(citySelectVal),
							"area": encodeURIComponent(areaSelectVal),
							"zip": postCodeVal,
							"address": encodeURIComponent(streetVal),
							"consignee": encodeURIComponent(nameVal),
							"mobile": mobileVal,
							"telphone":telphoneVal
						},
						type: 'post',
						success: function(data){
							if(data.success){
								var callId = data.logistics.id,
									callAddr = data.logistics.address,
									callPhone = data.logistics.phone;
								var dhtml = '<li class="addr-list">'+
												'<input type="radio" name="addressId" class="J_Change_Addr" value="'+callId+'">'+
													'<label class="label" for="label">'+ callAddr + 
														'<em class="tphone">'+callPhone+'</em>'+
													'</label>'+
											 '</li>';
								// 发请求 渲染配送公司
								var flag = userSameAddr(newAddrs,callId);
								
								//收起来
								//newAddrs.slideUp();
								if(flag){
									D.get('#address-list') && D.append(D.create(dhtml),D.get('#address-list'));
									var allInput = D.query('#address-list .J_Change_Addr'),
										lastInput = allInput[allInput.length - 1];
									lastInput.checked = true;
								}else{
									// 使用其他地址收起来
									newAddrs.slideUp();
									var allInput = D.query('#address-list .J_Change_Addr'),
										lastInput = allInput[allInput.length - 1];
									lastInput.checked = true;
								}
								
							}	
						}
					});
				});
			}
		},
		_linkage: function(tagVal){
			var self = this;
			var arr = [],
				obj = {};
			for(var list in self._tdist){
				if(self._tdist.hasOwnProperty(list)){
					if(self._tdist[list][1] === tagVal){
						obj = {name:self._tdist[list],val:list};
						arr.push(obj);
					}
				}
			}
			return arr;
		}
	};
	return Electrical;
});
