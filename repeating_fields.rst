***************************
Creating a repeating field
***************************

For any custom-developed content type, here are some simple guidelines to get you started on creating a variable-count repeating field for the editing form. (For example, a repeating field that allows limitless addition of URL/link title pairs.) 


Typically, the submitted values for such fields are stored one of two ways:

    1. Submitted as an array, serialized, and stored in a TEXT field in the database, or
    2. Stored in a separate custom hash/lookup table, with one record per submitted field group


We will not cover the specifics of how to store your data, but you should have enough information to determine the best course of action for your development.


Adding to the editing form
==========================

First, we need to create the initial repeating field that will appear by default when creating new content. We will use the URL and link name example to demonstrate.

Since a repeating field is essentially a group of fields that can be duplicated as a group, we first need to define the two fields in our group, plus a unique ID field (usually objectid) that we may need to edit, update, or delete existing repeating fields when storing them in a separate database table. If you are simply serializing the entire list into a single text field, you do not need the unique objectid.::

    // Create the sub-array which constitutes the "repeating group"
    $repeater = [];
    $repeater[] = array(
        'type'=>'hidden',
        'base_name'=>'objectid');
    $repeater[] = array(
        'label'=>_('URL'),
        'type'=>'text',
        'base_name'=>'url',
        'required'=>true);
    $repeater[] = array(
        'label'=>_('Link text'),
        'type'=>'text',
        'base_name'=>'link_text',
        'required'=>true);

	// Now create a new field in the regular form configuration array
	// add set it to be a "repeating_group" whose 'repeater'
	// is the array we just made
	$form_array[] = array(
		'label'=>_('Additional URLs'),
		'type'=>'repeating_group',
		'name'=>'additional_urls',
		'repeater'=>$repeater,
		'default'=>$current_urls);

For new content, this will create a single repeating group field, ready for input, and an "Add another repeater" link will automatically be appended. The link has no HREF; you are expected to use Javascript to attach a click handler to the link, following the instructions below.

We will be using similar configuration code in our editorial controller to handle adding more repeating group fields.

Handling the "Add another"
==========================

At the bottom of the editing script, or in a separate javascript file that is attached to the editing interface, you need some javascript to handle the "Add another"::

	<script type="text/javascript" charset="utf-8">
	// defer until doc ready
	jQuery(document).ready(function() {

		// Click handler to add another row of a repeating group
		jQuery(document).on("click",".add_repeating",function(e) {
			e.preventDefault();

			// Get any existing repeater blocks, and count them
			var base = jQuery(this).prevAll(".f_repeating");
			var excount = base.length;

			// Store a reference to "Add another" link itself
			var adder = jQuery(this);

			// This is a public endpoint, so an "add_listener" handler must be
			// added to the controller.inc file. BE SURE TO INCLUDE an appropriate
			// if (isAuthorized('Role')) wrapper around the code if needed
			let endpoint = js_g_url+"ajax/dated_posts/add_listener/?ct="+excount;
			// For a more secure version, customize the 
			// [content_type]_editorial.inc controller and use something like
			// let endpoint = js_g_url+"editor/ajax/DatedPost/add_listener/?ct="+excount;
			jQuery.getJSON(
				endpoint,
				function(r,status) {
					// The response r.data should contain a chunk of HTML (form elements)
					// built from a FormRenderer configuration array
					if (status == "success" && r.status == "success") {
					
						// Insert the new elements in the right location
						if (excount > 0) {
							base.first().after(r.data);
						} else {
							adder.before(r.data);
						}

					} else {
						// Display the error/failure
						jSheet(r.feedback.msg);
					}
				}
			)
		});
	
		// Auto-repeater-titling
		jQuery(document).on("blur",".f_repeating input[type=text]:first-of-type",function(e) {
			jQuery(this).closest(".f_repeating").find(".f_repeater_title").html(this.value);
		});
	
		// Remove a repeating element
		// NB: This simply removes the element from the DOM, and does not 
		// remove it from the database (which should happen when processing the form)
		jQuery(document).on("click",".repeater_remove",function(e) {
			e.preventDefault();
			jQuery(this).closest(".f_repeating").remove();
		});

	});
	</script>

Processing the editing form
==========================

If the main form prefix was "post_", and the name of our repeating_group field is "additional_urls," the elements containing the form fields  submitted for the repeating group will be an a nested array with the form

``[repeating-group-fieldname][index][name-of-sub-field]``


Or, following our example,::

	additional_urls[0][objectid]
	additional_urls[0][url]
	additional_urls[0][link_text]

and subsequent entries would be::

	additional_urls[1][objectid]
	additional_urls[1][url]
	additional_urls[1[link_text]

and so on. In fact, each of the field elements and its containers are given unique IDs and can be targeted with CSS or Javascript. Simply examine the rendered source code to get an idea of its consistent syntax.

For our demonstration, we have created a simple lookup table in MySQL called hash_content_urls with 4 fields::

	ref_class
	ref_id
	url
	link_text


	// Delete existing entries and rebuild from scratch (easiest way to handle updates)
	$oracle = new Oracle();
	$oracle->set_tablename('hash_content_urls');
	$oracle->set_where_clause("ref_class = 'DatedPost' AND ref_id = $this->objectid");
	$oracle->delete_record();


	// Loop over the submitted custom field array
	foreach($_POST[additional_urls] as $index=>$arr) {
		// Sanitize the individual fields
		$url = cleantext($arr['url'],'url');
		$link = cleantext($arr['link_text']); 
		// Reset the Oracle from the last iteration
		$oracle->reset();
		$oracle->set_set_clause("ref_class = 'DatedPost',ref_id = $this->objectid");
		$oracle->set_set_clause("url = '$url',link_text = '$link'");
		$oracle->insert();
	}
