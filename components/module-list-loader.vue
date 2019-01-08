<template>
	<div :class="name">
		<admin-list
			v-if="config"
			:title="config.title"
			:columns="config.columns"
			:actions="config.actions"
			:buttons="config.buttons"
			:data="data"
			@filter="filter"
			@edit="edit"
			@create="create"
			@remove="remove"
			@select="select"
		></admin-list>
	</div>
</template>

<script>
	import adminList from "./list.vue";
	import adminRouter from "../lib/router.js";
	import { validate } from "jsvalidator";
	
	export default {
		props : ["name", "routerArgs"],
		created : async function() {
			const module = await import(`@modules/${this.name}.js`);
			const data = module.default({ routerArgs : this.routerArgs });
			
			validate(data, {
				type : "object",
				schema : [
					{ name : "title", type : "string" },
					{ name : "columns", type : "array" },
					{ name : "createUrl", type : "string" },
					{ name : "idColumn", type : "string", default : "id" },
					{ name : "actions", type : "array" },
					{ name : "buttons", type : "array" },
					{
						name : "methods",
						type : "object",
						schema : [
							{ name : "getData", type : "function" },
							{ name : "remove", type : "function" }
						],
						allowExtraKeys : false
					}
				],
				allowExtraKeys : false,
				throwOnInvalid : true
			});
			
			this.config = data;
		},
		data : function() {
			return {
				config : undefined,
				data : []
			}
		},
		methods : {
			edit : function({ row }) {
				adminRouter.go({
					url : this.config.createUrl,
					type : "overlay",
					args : {
						filter : {
							[this.config.idColumn] : row[this.config.idColumn]
						},
						save : () => {
							adminRouter.back();
							adminRouter.go(this.routerArgs);
						}
					}
				});
			},
			filter : async function() {
				this.data = await this.config.methods.getData();
			},
			create : function() {
				adminRouter.go({
					url : this.config.createUrl,
					type : "overlay",
					args : {
						save : () => {
							adminRouter.back();
							adminRouter.go(this.routerArgs);
						}
					}
				});
			},
			remove : async function({ row }) {
				adminRouter.dialogOpen({
					title : "Delete?",
					text : "This will will permanently delete this item.",
					buttons : [
						{
							name : "cancel",
							type : "button",
							theme : "none",
							label : "Cancel",
							click : () => {
								adminRouter.dialogClose()
							}
						},
						{
							name : "remove",
							type : "button",
							theme : "destructive",
							label : "Delete",
							click : async () => {
								try {
									await this.config.methods.remove({ filter : { [this.config.idColumn] : row[this.config.idColumn] } });
								} catch(e) {
									adminRouter.errorDialog(e);
									return;
								}
								
								adminRouter.dialogClose();
								adminRouter.go(this.routerArgs);
							}
						}
					]
				});
			},
			select : function({ row }) {
				this.routerArgs.args.select({ row });
			}
		},
		components : {
			adminList
		}
	}
</script>