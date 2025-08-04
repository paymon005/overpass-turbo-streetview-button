// ==UserScript==
// @name         Overpass Turbo Street View Integration
// @namespace    http://tampermonkey.net/
// @version      1.1
// @description  Add "Load Street View" button to popups on overpass-turbo.eu
// @author       You
// @match        https://overpass-turbo.eu/*
// @match        http://overpass-turbo.eu/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    // Function to extract coordinates from popup
    function extractCoordinatesFromPopup(popup) {
        const coordElement = popup.querySelector('a[href^="geo:"]');
        if (coordElement) {
            const href = coordElement.getAttribute('href');
            const match = href.match(/geo:([+-]?\d+\.?\d*),([+-]?\d+\.?\d*)/);
            if (match) {
                return {
                    lat: parseFloat(match[1]),
                    lng: parseFloat(match[2])
                };
            }
        }
        return null;
    }

    // Function to create Street View button
    function createStreetViewButton(lat, lng) {
        const button = document.createElement('a');
        button.innerHTML = '🗺️ Street View';
        button.style.cssText = `
            display: inline-block;
            background: #4285f4;
            color: white !important;
            text-decoration: none !important;
            padding: 6px 12px;
            border-radius: 4px;
            font-size: 12px;
            margin-left: 8px;
            cursor: pointer;
            font-weight: bold;
            transition: background-color 0.2s;
        `;

        button.addEventListener('mouseenter', function() {
            button.style.background = '#3367d6';
        });

        button.addEventListener('mouseleave', function() {
            button.style.background = '#4285f4';
        });

        button.addEventListener('click', function(e) {
            e.preventDefault();
            const streetViewUrl = `https://www.google.com/maps/@${lat},${lng},3a,75y,90t/data=!3m6!1e1!3m4!1s0x0:0x0!2e0!7i13312!8i6656`;
            window.open(streetViewUrl, '_blank');
        });

        return button;
    }

    // Function to add Street View button to a popup
    function addStreetViewToPopup(popup) {
        // Check if button already exists
        if (popup.querySelector('.streetview-button')) {
            return;
        }

        const coords = extractCoordinatesFromPopup(popup);
        if (!coords) {
            return;
        }

        // Find the coordinates paragraph
        const coordParagraph = popup.querySelector('p');
        if (coordParagraph && coordParagraph.querySelector('a[href^="geo:"]')) {
            const button = createStreetViewButton(coords.lat, coords.lng);
            button.classList.add('streetview-button');

            // Add button after the geo link
            const geoLink = coordParagraph.querySelector('a[href^="geo:"]');
            const small = coordParagraph.querySelector('small');
            if (small) {
                coordParagraph.insertBefore(button, small);
            } else {
                coordParagraph.appendChild(button);
            }
        }
    }

    // Observer to watch for new popups
    function setupPopupObserver() {
        const observer = new MutationObserver(function(mutations) {
            mutations.forEach(function(mutation) {
                mutation.addedNodes.forEach(function(node) {
                    if (node.nodeType === 1) { // Element node
                        // Check if the added node is a popup
                        if (node.classList && node.classList.contains('leaflet-popup')) {
                            setTimeout(() => addStreetViewToPopup(node), 100);
                        }

                        // Check if the added node contains popups
                        const popups = node.querySelectorAll && node.querySelectorAll('.leaflet-popup');
                        if (popups) {
                            popups.forEach(popup => {
                                setTimeout(() => addStreetViewToPopup(popup), 100);
                            });
                        }
                    }
                });
            });
        });

        observer.observe(document.body, {
            childList: true,
            subtree: true
        });

        return observer;
    }

    // Function to process existing popups
    function processExistingPopups() {
        const existingPopups = document.querySelectorAll('.leaflet-popup');
        existingPopups.forEach(popup => {
            addStreetViewToPopup(popup);
        });
    }

    // Initialize the script
    function init() {
        console.log('Overpass Turbo Street View Integration loaded');

        // Process any existing popups
        processExistingPopups();

        // Set up observer for new popups
        setupPopupObserver();

        // Also check periodically for popups that might have been missed
        setInterval(processExistingPopups, 2000);
    }

    // Start initialization when page loads
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', init);
    } else {
        // If page is already loaded, wait a bit for the map to initialize
        setTimeout(init, 1000);
    }
})();