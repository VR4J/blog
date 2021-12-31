---
title: Contact
header: Connecting...
excerpt: Use this form below if you want to get in personal contact with me, and i'll make sure your personal information is handled with the utter most care.
date: 2021-12-31 09:28:48
---

<form class="mb-5" method="post" id="contact-form">


<input type="text" class="form-control error" required="true" name="name" id="name" placeholder="Name">
<input type="text" class="form-control error" required="true" name="email" id="email" placeholder="Email">
<input type="text" class="form-control" required="true" name="subject" id="subject" placeholder="Subject">
<textarea class="form-control" required="true" name="message" id="message" cols="30" rows="7" placeholder="Message"></textarea>

<div class="control-container">
    <input type="submit" value="Send Message" class="btn btn-primary rounded-0 py-2 px-4">
    <div class="g-recaptcha" data-sitekey="6Le3sd4dAAAAAF-bzYkrJggdMd0XuPtbo3EoL81_"></div>
</div>

<div id="captcha-validation" style="display: none; flex-direction: row; justify-content: end;">
    <p style="display: flex; color: red; font-size: 14px; align-self: end;">Please verify that you're not a robot.. (beep boop beep beep boop).</p>
</div>

<script src="https://www.google.com/recaptcha/api.js" async defer></script>
<script src="https://code.jquery.com/jquery-3.6.0.min.js" integrity="sha256-/xUj+3OJU5yExlq6GSYGSHk7tPXikynS7ogEvDej/m4=" crossorigin="anonymous"></script>

<script>
    let form = document.getElementById('contact-form');
    
    if(screen.width <= 500) {
        $(".g-recaptcha").attr("data-size", "compact")
        grecaptcha.reset()
    }

    form.onsubmit = (e) => {
        e.preventDefault();

        if(grecaptcha.getResponse().length) {
            $("#captcha-validation").css("display", "none")

            $("input[type='submit']").val("Submitting...")
            $("input[type='submit']").attr("disabled", true)
        
            $.ajax({
                url: "https://script.google.com/macros/s/AKfycbxH8WG1LKFCKLxqnogx_STwozCKsSr2LTVsRvBHoUOOFKqDYWnLXd7pJIUMP8HXphaiBw/exec",
                method: "POST",
                dataType: "json",
                data: $("#contact-form").serialize(),
                success: function(response) {
                    
                    if(response.result == "success") {
                        window.location.href = '/contact-success';
                    }
                    else {
                        window.location.href = '/contact-error';
                    }
                },
                error: function() {
                    window.location.href = '/contact-error';
                }
            })
        } else {
            $("#captcha-validation").css("display", "flex")
        }   
    }
</script>
